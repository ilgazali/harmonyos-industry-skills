---
id: PHOTO-19
title: Picture-in-picture video edit - two AVPlayers on two surfaces, merged with an ffmpeg overlay filter
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/19_video_edit_pip_window.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_edit_pip_window-0000002395437788
sample: huawei_industry_tree/18_photography/downloads/VideoEditPiPWindow.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [base, common, display, fileUri, fs, hilog, image, media, photoAccessHelper, promptAction, util, window, XComponent, PanGesture, PinchGesture, GestureGroup, AVImageGenerator, MP4Parser.ffmpegCmd, showAssetsCreationDialog]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-18-0006, HW-18-0052, HW-18-0053, HW-18-0054, HW-18-0055, HW-18-0091, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when you have to **compose two videos into one frame** — a small
inset video that the user can drag and pinch over a main video, previewed live and
then burned into a new file. The consumer version of this is a reaction video, a
commentary overlay, a screen recording with a webcam corner, a before/after clip.

The technique has three parts and each is separable. **Preview**: two
`XComponent` surfaces in one `Stack`, each fed by its own `AVPlayer` instance, the
inset one carrying `translate` and `scale` driven by an exclusive
`GestureGroup(PanGesture, PinchGesture)`. **Timeline**: an `AVImageGenerator`
walking the file one second at a time to build a thumbnail strip, with three
horizontal `List`s scrolled together by their `Scroller`s. **Export**: the
preview's vp offsets normalised to fractions and interpolated into an ffmpeg
`overlay=W*x:H*y` filter so the burned result matches what was on screen.

That last conversion is the piece worth studying — it is the only place where a
gesture-driven UI and a command-line video filter have to agree on a coordinate
system, and getting it wrong means the export does not match the preview.

**This sample leaks aggressively.** `HW-18-0054` and `HW-18-0006` both need
fixing before any of it ships.

## Feature checklist

- Home page: a 开始创作 card opens `PhotoViewPicker` filtered to video; the picked
  file is copied into `cacheDir` and the edit page opens automatically.
- Edit page: the main video plays in a full-width surface with a play/pause bar and
  a running `mm:ss / mm:ss` readout.
- The 画中画 button in the bottom bar opens the picker a second time; only one inset
  is allowed and a second attempt toasts.
- The inset video appears at half the screen width, draggable within a bounded box
  and pinchable to scale.
- A per-second thumbnail strip is built for each video and the two strips plus a
  time ruler scroll together as playback advances.
- A cover frame covers each surface until the user first presses play.
- 导出 merges the two files with the inset at the position and scale shown, then
  offers the result to the gallery.
- Short videos work; failed exports can be retried.

## Architecture

One `entry` module, `Navigation` + `NavDestination` routing through a singleton
router, and one 698-line page that carries the entire feature.

```
entry/src/main/ets
├── common/Constants.ets           AppStorage keys (Chinese literals), PiP geometry
├── components/Home.ets            landing tab: picker, mock cover grid, custom tab bar
├── entryability/EntryAbility.ets  light colour mode, AppStorage 'player', avoid areas
├── entrybackupability/
├── model
│   ├── BottomTabModel.ets         TAB_BUTTON_INFO + FUNCTION_BUTTON_INFO
│   └── CoverMockModel.ets         static grid data
├── pages
│   ├── EditPage.ets               the feature: surfaces, gestures, thumbnails, merge, save
│   └── HomePage.ets               @Entry, Navigation host + four bottom tabs
└── utils
    ├── AppRouter.ets              singleton NavPathStack wrapper
    ├── AVPlayer.ets               both players (static fields) + shared state machine
    ├── DisplayUtil.ets            display width/height
    └── PromptActionUtil.ets       the loading dialog
```

The documented tree matches the zip.

**The design decision worth copying** is that one `AVPlayer` class serves both
videos and the *only* difference between them is a boolean threaded through the
state machine:

```typescript
setAVPlayerCallback(avPlayer: media.AVPlayer, isBasic: boolean) {
  avPlayer.on('timeUpdate', (time: number) => {
    if (isBasic) { this.currentTime = time; }
  });
  avPlayer.on('durationUpdate', (time: number) => {
    if (isBasic) { AppStorage.setOrCreate('duration', time); }
  });
```

Both players share every state transition, and only the two clock callbacks are
gated. That is the right shape: in a picture-in-picture edit exactly one video owns
the timeline and the other follows, so exactly one player should be allowed to
publish time.

**The decision worth avoiding** is the storage: `avPlayerBasic` and `avPlayerPip`
are **static** fields on the class while the class is also instantiated once into
`AppStorage` as `'player'`. Every entry into `EditPage` overwrites both statics with
new players and nothing releases the old ones (`HW-18-0054`). The AppStorage keys are
also Chinese string literals (`'原视频路径'`, `'画中画视频路径'`) declared in
`Constants` — they work, but a typo in a literal key fails silently at runtime rather
than at compile time.

## Implementation steps

1. **Copy the picked video out of the picker URI into `cacheDir`** before doing
   anything with it. Picker URIs are grant-scoped; a sandbox copy is not.
2. **Route on the copy, not on the tap**: `@StorageLink(BASIC_FILE_PATH) @Watch('onChange')`
   pushes `EditPage` when the path becomes non-empty, so the page is never entered
   without a file.
3. **Give each video its own `XComponent` and its own player.** In `onLoad`, take the
   surface ID, hand it to the shared `AVPlayer` wrapper via `setSurfaceID`, then call
   `avPlayerFdSrc(name)` — the wrapper reads back the surface ID during the
   `initialized` state.
4. **Wrap pan and pinch in `GestureGroup(GestureMode.Exclusive, ...)`** so one gesture
   wins per interaction and a two-finger pinch is not also read as a drag.
5. **Accumulate the pinch across gestures**: `scaleValue = pinchValue * event.scale`
   during, `pinchValue = scaleValue` on end. `event.scale` is relative to the start of
   the current pinch.
6. **Clamp the drag** against `minX/maxX/minY/maxY` derived from the display width and
   the inset height, so the inset cannot be pushed fully off-frame.
7. **Build the thumbnail strip with `AVImageGenerator.fetchFrameByTime`** at one-second
   steps, using each file's **own** duration (`HW-18-0052`).
8. **Set the "ready" flag on the first successful frame** (`timeUs === 0`), not at
   exactly one second (`HW-18-0055`).
9. **Normalise the preview geometry to fractions** before building the ffmpeg command,
   correcting for the fact that `scale` grows around the component's centre while
   `overlay` positions by the top-left corner.
10. **Clear `outputPath` in the ffmpeg failure callback** and `.catch` the save
    (`HW-18-0053`).
11. **Clear the progress interval in `aboutToDisappear`** (`HW-18-0006`), and in the same
    place `release()` both players and close their fds before unlinking any file
    (`HW-18-0054`).

## Verified snippets

All snippets are from `VideoEditPiPWindow.zip`. Corrected forms are marked.

**The inset surface and its gestures — `entry/src/main/ets/pages/EditPage.ets`** (as shipped)

```typescript
if (this.showPip) {
  XComponent({ type: XComponentType.SURFACE, controller: this.xComponentController2 })
    .width(this.uiContext.px2vp(Constants.PIP_WIDTH))
    .height(Constants.PIP_HEIGHT)
    .zIndex(3)
    .onLoad(() => {
      this.surfaceId = this.xComponentController2.getXComponentSurfaceId();
      this.player.setSurfaceID(this.surfaceId);
      this.player.avPlayerFdSrc(Constants.AVPLAYER_NAME_PIP);
    })
    .translate({ x: this.offsetX, y: this.offsetY })
    .scale({ x: this.scaleValue, y: this.scaleValue })
    .gesture(
      GestureGroup(GestureMode.Exclusive,
        PanGesture()
          .onActionStart((event: GestureEvent) => {
            this.startX = this.offsetX;              // snapshot: offsetX is cumulative from here
            this.startY = this.offsetY;
          })
          .onActionUpdate((event: GestureEvent) => {
            this.offsetX = this.startX + event.offsetX;
            this.offsetY = this.startY + event.offsetY;
            if (this.offsetX <= this.minX) { this.offsetX = this.minX; }
            if (this.offsetX >= this.maxX) { this.offsetX = this.maxX; }
            if (this.offsetY <= this.minY) { this.offsetY = this.minY; }
            if (this.offsetY >= this.maxY) { this.offsetY = this.maxY; }
          }),
        PinchGesture({ fingers: 2 })
          .onActionUpdate((event: GestureEvent) => {
            this.scaleValue = this.pinchValue * event.scale; // 缩放比例累积
          })
          .onActionEnd((event: GestureEvent) => {
            this.pinchValue = this.scaleValue;       // commit, so the next pinch starts here
          })
      )
    );
}
```

**Three details carry the drag-and-pinch.** The `startX`/`startY` snapshot in
`onActionStart` is what makes `event.offsetX` — which is measured from the start of
*this* gesture — compose with the position left by previous gestures; without it
every new drag would teleport the inset back near the origin. `pinchValue` is the
same idea for scale, committed in `onActionEnd`, which is why `event.scale` (always
1.0 at the start of a pinch) accumulates instead of resetting. And
`GestureMode.Exclusive` is what stops a two-finger pinch from also feeding the pan
handler and translating the inset while it resizes.

`translate` and `scale` are render transforms: they do not change layout, so the
`Stack`'s `.clip(true)` still crops to the original 200 vp band and the inset cannot
escape the preview area even if the clamps were wrong. The cover `Image` above the
surface repeats the same transforms and the same gesture group verbatim — 40 lines
duplicated — because the cover has to move with the video it covers. Extracting an
`@Extend` or a shared builder would be the obvious cleanup.

**The thumbnail walk — same file** (corrected, see `HW-18-0052`, `HW-18-0055`)

```typescript
async testFetchFrameByTime(filePath: string, isBasic: boolean) {
  let avImageGenerator: media.AVImageGenerator = await media.createAVImageGenerator();
  let file = fs.openSync(filePath, fs.OpenMode.READ_ONLY);
  try {
    avImageGenerator.fdSrc = { fd: file.fd } as media.AVFileDescriptor;
    let queryOption = media.AVImageQueryOptions.AV_IMAGE_QUERY_NEXT_SYNC;
    let param: media.PixelMapParams = {
      width: this.uiContext.px2vp(DisplayUtil.getWidth()),
      height: Constants.BASIC_HEIGHT,
    };
    // FIX: shipped code bounds the loop with this.duration — the BASIC video's duration —
    // for both videos. Read this file's own duration first.
    let ownDuration = await this.durationOf(filePath);
    for (let timeUs = 0; timeUs < ownDuration * 1000; timeUs += 1000000) {
      let frame = await avImageGenerator.fetchFrameByTime(timeUs, queryOption, param);
      if (isBasic) {
        this.pixelMapBasicList.push(frame);
        if (timeUs === 0) {            // FIX: shipped code tests timeUs === 1000000
          this.showBasic = true;       // 获取到帧图后展示封面
        }
      } else {
        this.pixelMapPipList.push(frame);
        if (timeUs === 0) {            // FIX: same
          this.showPip = true;
        }
      }
    }
  } catch (error) {
    hilog.error(DOMAIN, 'testTag', 'Get frame with err: %{public}s', JSON.stringify(error as BusinessError));
  } finally {
    avImageGenerator.release();
    fs.closeSync(file);
  }
}
```

**One duration for two files is the bug that hides behind a demo asset.** The
loop bound is `this.duration`, which only the basic player ever writes — the
`durationUpdate` callback is gated on `isBasic`. When the inset video is shorter,
`fetchFrameByTime` is asked for frames past its end and the loop dies into the
`catch`; when it is longer, the strip is silently truncated. With two clips of
similar length nothing looks wrong, which is exactly why it survived.

The `timeUs === 1000000` test is the other trap: the flag that reveals the whole
inset UI is only set on the *second* iteration, so a video of one second or less
never displays at all (`HW-18-0055`). Setting it on the first successful frame is
both simpler and correct.

Two things the sample gets right and are worth keeping: `AV_IMAGE_QUERY_NEXT_SYNC`
asks for the next sync frame at or after the timestamp, which is what you want for a
scrub strip (an exact-time query would force a decode from the previous keyframe for
every thumbnail), and `PixelMapParams` downsamples during extraction so the strip
never holds full-resolution frames.

**Preview geometry into an ffmpeg overlay — same file** (corrected, see `HW-18-0053`)

```typescript
videoMerge() {
  let sandboxPath1 = AppStorage.get(Constants.BASIC_FILE_PATH) as string;
  let sandboxPath2 = AppStorage.get(Constants.PIP_FILE_PATH) as string;
  const DATE_STR = (new Date().getTime()).toString();
  this.outputPath = this.ctx.cacheDir + util.format('/%s%s', DATE_STR, 'merged.mp4');
  let callBack: ICallBack = {
    callBackResult: async (code: number) => {
      PromptActionUtil.closeDialog();
      if (code === 0) {
        this.uiContext.getPromptAction().showToast({ message: '导出成功' });
        await this.saveVideoToGallery(this.outputPath);
      } else {
        this.outputPath = '';           // FIX: absent — a failed merge poisons every later export
        this.uiContext.getPromptAction().showToast({ message: '导出失败' });
      }
    }
  };
  // inset height as a fraction of the main video's height
  let height: number = Math.floor(Constants.PIP_HEIGHT * this.scaleValue / Constants.BASIC_HEIGHT * 1000) / 1000;
  // scale grows around the centre; overlay positions the top-left corner
  let adjustX: number = this.offsetX - (this.pinchValue - 1) * this.uiContext.px2vp(Constants.PIP_WIDTH) / 2;
  let adjustY: number = this.offsetY - (this.pinchValue - 1) * Constants.PIP_HEIGHT / 2;
  let offsetX: number = Math.floor(adjustX / this.uiContext.px2vp(DisplayUtil.getWidth()) * 1000) / 1000;
  let offsetY: number = Math.floor(adjustY / Constants.BASIC_HEIGHT * 1000) / 1000;
  let ffmpegCmd = `ffmpeg -i ${sandboxPath1} -i ${sandboxPath2} -filter_complex` +
    ` "[1]scale=-1:ih*${height}[pip];[0][pip]overlay=W*${offsetX}:H*${offsetY}" -c:a copy ${this.outputPath}`;
  MP4Parser.ffmpegCmd(ffmpegCmd, callBack);
}
```

**Everything becomes a fraction, and that is the whole trick.** The preview is
laid out in vp against a 200 vp band; the source files are whatever resolution the
camera produced. Dividing every measurement by its container — `PIP_HEIGHT /
BASIC_HEIGHT` for size, `adjustX / displayWidth` for position — produces numbers
that are resolution-independent, and `overlay=W*x:H*y` multiplies them back up by
the *main video's* real dimensions. `scale=-1:ih*h` keeps the inset's own aspect
ratio (`-1` means "derive the width"), so the fraction only has to control height.

The `adjustX`/`adjustY` correction is the part that is easy to miss. ArkUI's
`scale` transform grows a component about its centre while leaving its layout box
alone, so a pinched inset visually extends half the growth to the left and above its
untransformed origin. ffmpeg's `overlay` takes the top-left corner. Subtracting
`(pinchValue - 1) * size / 2` converts one convention to the other. Note it uses
`pinchValue` — the committed scale — while the height fraction uses `scaleValue`;
those agree only after `onActionEnd` has fired, so exporting mid-pinch shifts the
inset.

`-c:a copy` copies the audio stream untouched, which means the merged file carries
the **main** video's audio only — the inset is silent. That is the usual choice for
a reaction overlay, but it is a decision, not a default.

**Teardown — same file** (corrected, see `HW-18-0006`, `HW-18-0054`)

```typescript
aboutToDisappear(): void {
  clearInterval(this.proFreshTimer);                     // FIX: absent — timer polls a dead component
  this.player.releaseAll();                              // FIX: method to add — release() both static
                                                         // players and close their fds before any unlink
  const paths = [
    AppStorage.get(Constants.BASIC_FILE_PATH) as string,
    AppStorage.get(Constants.PIP_FILE_PATH) as string,
    this.outputPath
  ];
  for (const p of paths) {
    if (p !== '' && fs.accessSync(p)) {                  // FIX: sample unlinks unguarded and throws
      fs.unlinkSync(p);
    }
  }
  AppStorage.set(Constants.BASIC_FILE_PATH, '');
  AppStorage.set(Constants.PIP_FILE_PATH, '');
  this.pixelMapBasicList.length = 0;
  this.pixelMapPipList.length = 0;
  this.duration = 0;
  this.outputPath = '';
}
```

**Order matters here.** The shipped version unlinks the backing files while two
orphaned players still hold open fds to them, which on top of the leak means the
next session can start against a half-torn-down state. Release the players first,
then close the descriptors, then unlink — and guard each unlink with `accessSync`,
because the shipped code throws out of `aboutToDisappear` whenever a path is set but
the file is already gone (after a successful save, which unlinks the output itself),
aborting the rest of the teardown.

The progress timer is the cheapest of the three fixes and the most obviously wrong:
`setInterval` in `aboutToAppear`, no `clearInterval` anywhere in the file, so leaving
the edit page leaves a 100 ms poll running on a detached component for the process
lifetime (`HW-18-0006`).

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`, which is correct and
notable: reading uses `PhotoViewPicker` and writing uses
`showAssetsCreationDialog`, both permission-free. Compare `PHOTO-18` and `PHOTO-20`,
which run equivalent flows and still declare `WRITE_IMAGEVIDEO` (`HW-18-0004`).

`routerMap` is `$profile:route_map` and navigation goes through `AppRouter`, a
singleton wrapping one `NavPathStack`. `EntryAbility` forces `COLOR_MODE_LIGHT`
application-wide and publishes `topRectHeight`/`bottomRectHeight` from
`TYPE_SYSTEM` and `TYPE_NAVIGATION_INDICATOR` avoid areas into `AppStorage`.
`@ohos/mp4parser` is the only third-party dependency and it bundles ffmpeg.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The preview band is a hardcoded 200 vp** (`BASIC_HEIGHT`), and the inset is
  `displayWidth / 2` by 100 vp. Every fraction in `videoMerge` is derived from those
  constants, so the export matches the preview only while the preview keeps that
  exact geometry. A responsive layout must measure instead.
- Only one inset is supported — `pipNumber` and the `filePath !== ''` guard enforce it.
- `-c:a copy` requires the main video's audio codec to be container-compatible with
  MP4; a source with, say, PCM audio will fail the merge.
- The thumbnail strip is one PixelMap per second held in an `@State` array with no
  cap. A ten-minute video is 600 decoded frames in memory.
- 剪辑 / 回退 / 前进 in the play bar and three of the four bottom tabs are toasts
  reading 请开发者自行实现该功能 ("implement this yourself").
- `saveVideoToGallery` indexes `DES_FILE_URIS[0]` only after a length check, but does
  not handle the user dismissing the creation dialog beyond skipping the copy silently.

## Pitfalls

- **`HW-18-0006` (B/low, confirmed) — the progress interval is never cleared.**
  `setInterval` in `aboutToAppear` polls `player.currentTime` every 100 ms; there is
  no `clearInterval` in the file and no `aboutToDisappear` hook for it. Leaving the
  page leaves the timer running on a detached component. Fix: `clearInterval(this.proFreshTimer)`
  in `aboutToDisappear`.
- **`HW-18-0052` (B/medium, confirmed) — the inset's thumbnail loop uses the main
  video's duration.** `testFetchFrameByTime` bounds on `this.duration`, which only the
  basic player writes. A shorter inset has frames queried past its end (failed
  thumbnails, loop aborts into the catch); a longer one is truncated. Fix: read each
  file's own duration from its metadata before looping.
- **`HW-18-0053` (B/medium, confirmed) — a failed merge poisons every later export.**
  `videoMerge` sets `outputPath` before ffmpeg runs, and the `code !== 0` branch never
  clears it. The export button then takes the `else` path forever, calling
  `saveVideoToGallery` un-awaited and un-caught on a file ffmpeg never produced.
  Fix: reset `outputPath` in the failure callback and `.catch` the save.
- **`HW-18-0054` (B/medium, confirmed) — players and fds are never released, and
  teardown unlinks files they still hold.** `avPlayerFdSrc` creates a player and opens
  an fd per entry into the static `avPlayerBasic`/`avPlayerPip`, overwriting whatever
  was there; no `release()` or `close()` exists anywhere. `aboutToDisappear` then
  `unlinkSync`s the backing files unguarded, throwing and aborting the rest of the
  teardown whenever a path is set but the file is already gone. Fix: release both
  players, close the fds, guard the unlinks with `access()`.
- **`HW-18-0055` (B/low, confirmed) — videos of one second or less never appear.**
  `showBasic`/`showPip` are set only when the frame loop hits exactly
  `timeUs === 1000000`; a `≤1 s` clip runs only the `timeUs = 0` iteration, so the
  flag never flips and the `if (this.showBasic)` / `if (this.showPip)` blocks that
  render the surfaces and the strip never build. Fix: set the flag at `timeUs === 0`.
- **Beyond the filed findings:** the inset's cover `Image` duplicates the entire
  40-line gesture group of the surface it covers; `videoMerge` mixes `scaleValue` and
  `pinchValue` in the same calculation, so an export triggered mid-pinch is offset;
  and the two Chinese-literal AppStorage keys in `Constants` give no compile-time
  protection against a typo.

## References

- `huawei_industry_tree/18_photography/docs/19_video_edit_pip_window.md` — the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_edit_pip_window-0000002395437788
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` — the state machine, `url`, `surfaceId`, `timeUpdate`, `durationUpdate`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/04_media/arkts-apis-media-AVImageGenerator.md` — `fetchFrameByTime`, `AVImageQueryOptions`, `PixelMapParams`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avimagegenerator
- `documentation/harmonyos-guides/05_media/avimagegenerator.md` — the thumbnail extraction flow and why the fd must outlive it
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/avimagegenerator
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` — `XComponentType.SURFACE` and `getXComponentSurfaceId`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` — `offsetX`/`offsetY` measured from gesture start
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pinchgesture.md` — `event.scale` relative to the pinch start
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pinchgesture
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` — `select` and `PhotoViewMIMETypes.VIDEO_TYPE`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` — `showAssetsCreationDialog`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` — `NavPathStack`, `NavDestination`, `routerMap`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `PHOTO-17` — the same mp4parser/ffmpeg export path, for an aspect-ratio crop
