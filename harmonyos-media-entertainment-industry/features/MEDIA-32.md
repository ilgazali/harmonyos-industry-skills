---
id: MEDIA-32
title: Video screenshot with frame stepping - snapshot the XComponent, then nudge it a frame at a time
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/32_video_screenshot.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_screenshot-0000002368729597
sample: huawei_industry_tree/13_media_entertainment/downloads/VideoScreenshot.zip
kits: ["@kit.AVSessionKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.BackgroundTasksKit", "@kit.BasicServicesKit", "@kit.ImageKit", "@kit.MediaKit", "@kit.PerformanceAnalysisKit"]
apis: ["UIContext.getComponentSnapshot", "componentSnapshot.get", XComponent, XComponentController, "media.createAVPlayer", "AVPlayer.seek", "media.SeekMode", "AVPlayer.setSpeed", "avSession.createAVSession", setAVMetadata, setAVPlaybackState, "backgroundTaskManager.startBackgroundRunning", "wantAgent.getWantAgent", "resourceManager.getRawFdSync", "@StorageLink", "@Watch", display, image, systemDateTime]
permissions: [ohos.permission.INTERNET]
min_api: 20
modules: [entry]
findings: [HW-13-0072, HW-13-0073, HW-13-0074, HW-13-0045, HW-13-0046, HW-13-0012, HW-13-0003, HW-13-0099]
status: verified-with-fixes
---

## When to use

Load this card when the user must be able to **capture the exact frame they
meant**. A tap on a screenshot button lands wherever the decoder happened to be;
the fix is a two-stage flow - grab a frame, then let the user walk backwards and
forwards one frame at a time on a review screen until the picture is right.

The capture technique is `componentSnapshot` against the `XComponent`'s id, not
a pixel readback from the player. That is the transferable part: any surface a
component renders - a video, a canvas, a map, a chart - can be turned into a
`PixelMap` by giving the component an `.id()` and asking the snapshot service
for it. The stepping technique is the second: `seek` to
`screenshotTime ± 1000 / frameRate` with `media.SeekMode.SEEK_CLOSEST`, wait for
the surface to repaint, then snapshot again.

The sample is also a full `AVSession` integration - system playback controls,
metadata with a cover image, a continuous task for background audio - which is
where three of its defects live. **Treat the session code as a checklist of what
to get right, not as code to copy**: the seek command is misinterpreted
(`HW-13-0072`), the cover `PixelMap` is released before use (`HW-13-0073`), and
the background task is started without the permission that allows it
(`HW-13-0074`).

## Feature checklist

- A landscape full-screen player over an `XComponent` surface, playing the
  videos bundled under `resources/rawfile/video`.
- A control bar with play/pause, next video, screenshot, a seek slider and
  elapsed/total times.
- Long-pressing the right half of the screen plays at 2x and shows a 倍速播放中
  (playing at speed) pill; releasing returns to the speed chosen in the menu.
- A speed menu offering 0.5x, 1x, 1.5x, 2x and 3x, each confirmed by a pill that
  fades out after two seconds.
- The screenshot button pauses playback, captures the surface, and animates the
  captured image down into a thumbnail in the bottom-left corner.
- Tapping the thumbnail opens a review page showing the capture full size, with
  上一帧 / 下一帧 (previous / next frame) buttons on either side.
- Each frame step seeks the player by one frame and refreshes the shown image;
  taps within 600 ms of each other are ignored.
- Share and save buttons on the review page raise "demo only" toasts.
- The player registers an `AVSession` so the system playback controls can drive
  it, and requests a continuous task so audio survives backgrounding.

## Architecture

One `entry` module, two pages, one logger. `VideoPage` is 928 lines and holds
the player, the session, the gestures and the capture; `ScreenshotPage` is the
review screen.

```
entry/src/main/ets
├── common/Constants.ets            percentages, sizes, FRAME_RATE = 25, the two toast strings
├── entryability/EntryAbility.ets
├── pages/
│   ├── ScreenshotPage.ets          NavDestination: the capture, prev/next frame, share/save (112 lines)
│   └── VideoPage.ets               @Entry: AVPlayer + AVSession + XComponent + all the controls
└── util/logger.ets                 hilog wrapper
```

The documented tree spells the logger `util/Logger.ets`; the zip ships
`util/logger.ets` (`HW-13-0003`). The projects enable `caseSensitiveCheck`, so
the documented path does not resolve.

**The design decision worth copying** is that the two pages do not share
objects, they share `AppStorage` cells - and they share them in **both**
directions:

```typescript
// VideoPage
@StorageLink('previousFrame') @Watch('clickPreviousFrame') previousFrame: boolean = false;
@StorageLink('nextFrame')     @Watch('clickNextFrame')     nextFrame: boolean = false;
// ...
AppStorage.setOrCreate('img', this.pixmap);

// ScreenshotPage
@StorageLink('previousFrame') previousFrame: boolean = false;
this.previousFrame = !this.previousFrame;      // the request
this.img = AppStorage.get('img') as image.PixelMap;   // the answer, 600 ms later
```

`ScreenshotPage` is pushed on the navigation stack over a `VideoPage` that stays
alive, so the player it must drive is on the page underneath. Rather than pass a
controller down, the review page **toggles a boolean** and `@Watch` on the
player page turns the toggle into a seek. The boolean is a signal, not a state -
its value is meaningless, only its changes matter. That is a clean way to send a
command upstream across a `NavDestination` boundary without a shared service
object, and it is worth copying whenever the pushed page is a satellite of the
page beneath it.

The price is that the round trip is timed rather than acknowledged: the player
snapshots 500 ms after the seek, the review page re-reads `img` after 600 ms,
and a 600 ms tap debounce keeps the two from overlapping. Three magic numbers
hold the handshake together. A `seekDone`-driven `AppStorage` write would remove
all three.

## Implementation steps

1. **Give the `XComponent` an id** (`.id('xComponent')`) and set its surface
   size in `onLoad`, then hand `getXComponentSurfaceId()` to the player as
   `avPlayer.surfaceId` in the `initialized` state.
2. **Feed the player from `rawfile`**: `getRawFileListSync('video')` for the
   list, `getRawFdSync` per entry, and build `media.AVFileDescriptor` from the
   `{fd, offset, length}` triple. **Pair every `getRawFd` with `closeRawFd`**
   after releasing the player (`HW-13-0012`).
3. **Drive everything from `on('stateChange')`**: assign `fdSrc` in `idle`, set
   the surface and `prepare()` in `initialized`, read `duration` and start the
   session in `prepared`, and mirror each state into the `AVPlaybackState`.
4. **Capture with the snapshot service from the `UIContext`**:
   `this.getUIContext().getComponentSnapshot().get('xComponent')`. The document's
   snippet uses the bare global `componentSnapshot.get`, which is the form the
   UIContext migration replaces.
5. **Pause before capturing.** The screenshot handler sets `isPlay = false` and
   calls `avPlayer.pause()` first, then records `screenshotTime = currentTime`
   as the anchor every later frame step is relative to.
6. **Step by `1000 / FRAME_RATE` ms with `SeekMode.SEEK_CLOSEST`.** `SEEK_CLOSEST`
   is the only mode that can land on a non-key frame, which is exactly what
   single-frame stepping needs.
7. **Signal across pages with a toggled boolean plus `@Watch`**, and debounce
   the taps with `systemDateTime.getTime(false)`.
8. **Initialise the speed index to the real starting speed** - `1`, not `2`
   (`HW-13-0045`).
9. **Track the "wait until prepared" interval in a field and clear it on
   teardown and on error** (`HW-13-0046`).
10. **Treat the session's `seek` argument as an absolute position in ms**
    (`HW-13-0072`); it is not a relative offset in seconds.
11. **Keep the cover `PixelMap` alive** until the metadata that references it is
    no longer needed (`HW-13-0073`).
12. **Declare `ohos.permission.KEEP_BACKGROUND_RUNNING`** before calling
    `startBackgroundRunning` (`HW-13-0074`).

## Verified snippets

All snippets are from `VideoScreenshot.zip`. Corrected forms are marked.

**The capture and the hand-off — `entry/src/main/ets/pages/VideoPage.ets`** (as shipped)

```typescript
Image($r('app.media.screenshot'))
  .width('26vp')
  .height('26vp')
  .onClick(async () => {
    this.isPlay = false;
    this.avPlayer?.pause();
    this.screenshotTime = this.currentTime;      // the anchor for every later frame step
    this.screenshot();
    this.showImg = true;

    this.getUIContext().animateTo({
      duration: 500,
      iterations: 1,
      playMode: PlayMode.Normal,
    }, () => {
      this.imgWidth = 375;
      this.imgHeight = 166;
      this.imgpositionY = this.getUIContext().px2vp(this.windowHeight) - this.imgHeight;
    });
  });

async screenshot() {
  await this.getUIContext().getComponentSnapshot().get('xComponent').then((pixmap: image.PixelMap) => {
    this.pixmap = pixmap;
  });
  AppStorage.setOrCreate('img', this.pixmap);
}
```

**`componentSnapshot.get(id)` is a render-tree read, not a player API.** It
takes the component's id and returns a `PixelMap` of what that component is
currently displaying - which is why the `XComponent` needs `.id('xComponent')`
and why `avPlayer.pause()` comes first: a running surface can repaint between
the tap and the capture. Because it goes through the render tree, the same call
works for a `Canvas` or a `Web` just as well as for video.

The animation is the whole feedback design. The captured image is displayed
full-screen at `zIndex(5)` and then animated - width, height and Y position
together - down to a 375x166 thumbnail at the bottom of the screen, so the user
sees *the frame they took* shrink into the corner rather than a generic toast.
Tapping the thumbnail restores the geometry and pushes `screenshotPage`.

**Frame stepping — same file, plus `ScreenshotPage.ets`** (as shipped)

```typescript
// VideoPage: the two @Watch handlers
@StorageLink('previousFrame') @Watch('clickPreviousFrame') previousFrame: boolean = false;
@StorageLink('nextFrame') @Watch('clickNextFrame') nextFrame: boolean = false;

async clickPreviousFrame() {
  this.avPlayer?.seek(this.screenshotTime - 1000 / Constants.FRAME_RATE, media.SeekMode.SEEK_CLOSEST);
  setTimeout(() => {
    this.screenshot();
  }, 500);
  this.screenshotTime -= 1000 / Constants.FRAME_RATE;
}

// ScreenshotPage: the request side
.onClick(() => {
  let time = systemDateTime.getTime(false);
  if (time - this.lastTime < 600) {
    return;                                   // debounce: one step per 600 ms
  }
  this.lastTime = time;
  this.previousFrame = !this.previousFrame;   // the signal - the value carries no meaning
  setTimeout(() => {
    this.img = AppStorage.get('img') as image.PixelMap;
  }, 600);
});
```

**`SEEK_CLOSEST` is the load-bearing choice.** The other modes
(`SEEK_PREV_SYNC`, `SEEK_NEXT_SYNC`) snap to the nearest sync frame, which on a
typical GOP is up to two seconds away - stepping by 40 ms with either of them
would either not move at all or jump a whole GOP. `SEEK_CLOSEST` decodes to the
requested position, which costs more but is the only mode that makes single-frame
navigation meaningful.

`1000 / Constants.FRAME_RATE` with `FRAME_RATE = 25` is 40 ms per step - a
constant, not something read from the media. A 30 fps or 60 fps clip steps by
the wrong amount, and stepping accumulates: `screenshotTime` is adjusted by the
same nominal 40 ms each time, so after several steps the anchor and the decoder
can drift apart. Reading the real frame rate from the track description and
anchoring on `avPlayer.currentTime` after `seekDone` would remove both problems.

The 500 ms in `clickPreviousFrame` and the 600 ms in `ScreenshotPage` are the
same handshake seen from the two ends: the player must have finished seeking and
repainted before the snapshot, and the snapshot must be in `AppStorage` before
the review page reads it. `avPlayer.on('seekDone')` is already registered in
this file and is the event both timers are standing in for.

**The AVSession seek command — same file** (corrected, see `HW-13-0072`)

```typescript
// 跳转节点监听事件 - the system playback controller asks for a position
curAvSession.on('seek', (time: number) => {
  logger.info(`AVPlayerVideo AVSession on seek entry time : ${time}`);
  this.seekTo(time);                 // FIX: the sample calls this.setSeek(time)
});

// FIX: new - the absolute path the session command needs
seekTo(positionMs: number) {
  if (!this.avPlayer || this.OPERATE_STATE.indexOf(this.avPlayer.state) === -1) {
    logger.error('AVPlayerVideo seekTo failed. no avPlayer or state is not prepared/playing/paused/completed');
    return;
  }
  const target = Math.min(Math.max(positionMs, 0), this.avPlayer.duration);
  this.currentTime = target * this.durationTime / this.avPlayer.duration;
  this.currentStringTime = this.secondToTime(Math.floor(target / 1000));
  this.avPlayer.seek(target, media.SeekMode.SEEK_PREV_SYNC);
}

// the relative helper, unchanged - used by the app's own ±N-second controls
setSeek(addSecond: number, curTime?: number) {
  if (!curTime) {
    curTime = this.avPlayer!.currentTime;
  }
  let curMillSeconds: number = curTime + addSecond * 1000;
  curMillSeconds = Math.min(Math.max(curMillSeconds, 0), this.avPlayer!.duration);
  // ...
  this.avPlayer!.seek(curMillSeconds, media.SeekMode.SEEK_PREV_SYNC);
}
```

**`AVSession`'s `seek` callback delivers an absolute position in milliseconds.**
`setSeek` interprets its argument as a relative offset **in seconds** and
computes `currentTime + addSecond * 1000`. Feeding a position of, say, 90000 ms
into it asks for `currentTime + 90000000` ms, which the clamp turns into the end
of the video - so dragging the progress bar in the system playback controller
always jumps to the end. The two units are why the fix is a separate entry point
rather than a flag: one caller means "go here", the other means "move by this
much". `HW-13-0072` records the identical defect in the `Mirror` sample, which
copied the same file.

**The session cover image — same file** (corrected, see `HW-13-0073`)

```typescript
async saveRawFileToPixelMap(rawFilePath: string) {
  let value: Uint8Array = await this.getUIContext().getHostContext()!
    .resourceManager.getRawFileContent(rawFilePath);
  let imageBuffer: ArrayBuffer = value.buffer as ArrayBuffer;
  let imageSource: image.ImageSource = image.createImageSource(imageBuffer);
  await imageSource.createPixelMap({ desiredSize: { width: 900, height: 900 } })
    .then((pixelMap: image.PixelMap) => {
      this.curPixelMap = pixelMap;
      // FIX: the sample calls pixelMap.release() here, on the object it just stored
      imageSource.release();
    }).catch(() => {
      logger.error('AVPlayerVideo Failed to create pixelMap object through image decoding parameters.');
    });
}

// consumed later, in generateAVMetadata():
//   mediaImage: this.curPixelMap,
```

**`curPixelMap` is stored and then destroyed in the next statement.**
`this.curPixelMap = pixelMap; pixelMap.release();` leaves the field pointing at a
released native object, and `generateAVMetadata` hands that to `setAVMetadata`
as `mediaImage` - so the playback centre gets no cover art, or an error. The
`imageSource.release()` on the following line is correct and should stay: the
source is finished once the `PixelMap` exists. The `PixelMap` itself must live
until the metadata that references it is replaced. The sibling `Mirror` sample
keeps it alive and is the correct version of the same function.

Note that this cover is a static bundled `first.png`, not the video's own first
frame - a real player would generate it with `AVImageGenerator`, or reuse the
snapshot machinery this sample already has.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.KEEP_BACKGROUND_RUNNING",     // FIX: absent in the sample
    "reason": "$string:background_reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
],
"abilities": [
  {
    "name": "EntryAbility",
    "orientation": "landscape",
    "backgroundModes": ["audioPlayback"]
  }
]
```

- The shipped manifest declares `ohos.permission.INTERNET` - which nothing in
  the sample uses, every source being a bundled rawfile - and **not**
  `KEEP_BACKGROUND_RUNNING`, which `startContinuousTask` needs.
  `backgroundTaskManager.startBackgroundRunning` therefore fails with `201`
  (permission denied) on every launch, and the advertised background playback
  never starts (`HW-13-0074`). The sibling `Mirror` sample declares it correctly.
- `backgroundModes: ["audioPlayback"]` on the ability is required as well, and
  is present - it declares *which* background mode the continuous task may use;
  it does not grant the right to start one.
- `startContinuousTask` is called from `startAVSession`, i.e. as soon as the
  player reaches `prepared`, and `stopContinuousTask` from `aboutToDisappear`.
  That pairing is right; only the permission is missing.
- `"orientation": "landscape"` pins the player, which is why the snapshot
  geometry (`windowWidth` / `windowHeight` from `display.getDefaultDisplaySync`)
  can be treated as fixed.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `SEEK_CLOSEST` is more expensive than a sync-frame seek and its cost depends
  on the GOP length; on a long-GOP clip a frame step can take noticeably longer
  than the 500 ms the sample waits before snapshotting.
- `Constants.FRAME_RATE` is a hardcoded `25`. Clips at any other rate step by the
  wrong interval.
- The review page reads its image from `AppStorage.get('img')`, so the two pages
  are coupled through a global cell - only one capture can be in flight, and
  nothing clears the cell when the pages close.
- `screenshot()` overwrites `this.pixmap` on every capture without releasing the
  previous `PixelMap`; the old ones are left to the GC.
- Share and save are toasts (`分享成功,此功能仅供演示` /
  `保存成功,此功能仅供演示`). There is no `SaveButton` and no photo-access flow -
  saving a capture to the gallery is not demonstrated.
- `stopPlay` calls `stop()`, then `reset()`, then `seek()` - a seek on a player
  that was just reset to `idle` has nothing to seek.

## Pitfalls

- **`HW-13-0072`** (B/medium, confirmed, systematic across two samples): the
  `AVSession` `seek` callback delivers an absolute position in ms, but is passed
  to `setSeek`, which treats its argument as a relative offset in seconds
  (`currentTime + time * 1000`). Every seek from the system playback controller
  jumps to the end of the video. Fix: give the session command its own absolute
  seek path.
- **`HW-13-0073`** (B/medium, confirmed): `saveRawFileToPixelMap` stores the
  cover `PixelMap` in `curPixelMap` and releases it on the next line, so
  `setAVMetadata` receives a released native object as `mediaImage`. Fix: drop
  the premature `release()`; keep the `imageSource.release()`.
- **`HW-13-0074`** (E/medium, confirmed): `startBackgroundRunning` is called
  without `ohos.permission.KEEP_BACKGROUND_RUNNING` being declared, while an
  unused `INTERNET` is. The call fails with `201` every time. Fix: declare the
  permission (and drop `INTERNET`).
- **`HW-13-0045`** (B/medium, confirmed, systematic across two samples):
  `speedIndex` is initialised to `2`, so the first long-press "restores" 2x on
  release instead of the actual 1x - before the user has ever opened the speed
  menu, a single long press leaves playback permanently at double speed. Fix:
  initialise `speedIndex = 1`.
- **`HW-13-0046`** (B/low, confirmed, systematic across three samples):
  `iconOnclick` starts a 100 ms `setInterval` to wait for `flag` (prepared) in a
  local variable, cleared only on success. A failed prepare leaves a 10 Hz timer
  running forever, and each tap spawns another, all of which later call `play()`.
  Fix: keep the id in a field, clear it in `aboutToDisappear` and on error, and
  guard against a second instance.
- **`HW-13-0012`** (B/low, confirmed, systematic across seven samples):
  `initFiles` calls `resourceManager.getRawFdSync` for every video in
  `rawfile/video` and never calls `closeRawFd`, so the raw HAP descriptors are
  held for the process lifetime. Fix: pair every `getRawFd` with `closeRawFd`
  after the player is released.
- **`HW-13-0003`** (E/low, confirmed, systematic): the documented tree lists
  `util/Logger.ets`; the zip ships `util/logger.ets`, and the project enables
  `caseSensitiveCheck`. Fix: regenerate the tree from the zip.

## References

- `huawei_industry_tree/13_media_entertainment/docs/32_video_screenshot.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_screenshot-0000002368729597
- `documentation/harmonyos-references/02_application-framework/js-apis-arkui-componentsnapshot.md` - `get`, and the `UIContext` form that replaces the global
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-arkui-componentsnapshot
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - `XComponentController`, `getXComponentSurfaceId`, `setXComponentSurfaceRect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - the state machine, `seek`, `SeekMode`, `seekDone`, `setSpeed`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/04_media/js-apis-avsession.md` - `createAVSession`, `setAVMetadata`, `setAVPlaybackState`, the `seek` and `setSpeed` commands
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-avsession
- `documentation/harmonyos-guides/03_application-framework/continuous-task.md` - `startBackgroundRunning`, `backgroundModes` and the permission it needs
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/continuous-task
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFdSync` and `closeRawFd`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `MEDIA-17` - the same speed menu and long-press gesture, and the shared `HW-13-0045` / `HW-13-0046`
- `MEDIA-29` - the same "hold for 2x" interaction built on a plain `Video` component instead
