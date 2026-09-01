---
id: PHOTO-17
title: Video aspect-ratio crop - a draggable border over an XComponent, cut with an ffmpeg crop filter
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/17_video_cropping.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_cropping-0000002385796378
sample: huawei_industry_tree/18_photography/downloads/VideoCropping.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MediaKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [common, display, fileIo, fileUri, hilog, media, photoAccessHelper, window, XComponent, XComponentController, PanGesture, MP4Parser.ffmpegCmd, showAssetsCreationDialog]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-18-0047, HW-18-0048, HW-18-0049, HW-18-0022, HW-18-0091, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when the user must **choose a rectangle over a playing video** —
a crop frame with ratio presets (原图, 9:16, 16:9, 1:1, 4:3, 3:4) and draggable
edges — and the chosen rectangle must then be applied to the file, not just to
the preview.

The transferable idea is the overlay: the crop frame is not four draggable
handles and not a second canvas, it is a single empty `Column` stretched over the
player whose **border widths** are the four margins of the crop. Set
`border.left = 40` and the left 40 vp go dark; the hole in the middle is the crop.
One `PanGesture` on that `Column` moves whichever edge the finger started near.
The same trick works for an image crop, a redaction box, a chart brush, or any
"dim everything except this rectangle" interaction.

The second half is the export path, and it is where the sample is weakest. Cutting
happens in `@ohos/mp4parser` through a literal ffmpeg command string, and saving
goes through `showAssetsCreationDialog`, which needs no album permission. Both are
worth copying; the file bookkeeping around them (`HW-18-0047`, `HW-18-0048`) is not.

## Feature checklist

- The page opens playing a bundled `flower.mp4` inside an `XComponent` surface.
- Six ratio chips under the video; the selected one turns blue.
- Tapping 原图 resets the frame to the whole video; tapping any other preset jumps
  the frame to that ratio's rectangle.
- Dragging near an edge of the frame moves that edge; the frame is clamped to the
  video bounds on all four sides.
- The 导出 (export) button runs an ffmpeg crop over the source file and writes the
  result to the sandbox.
- On success the result is offered to the gallery through the system creation
  dialog and a toast confirms.
- Exporting twice in a row works.
- The back arrow leaves the page.

## Architecture

One `entry` module, six files, no model layer beyond an `AVPlayer` wrapper class.

```
entry/src/main/ets
├── constants/StyleConstants.ets     layout numbers + ASPECT_RATIO = 16/9
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model/AVPlayerDemo.ets           AVPlayer state machine, url-from-fd playback
├── pages/MainPage.ets               @Entry: avoid-area padding, hosts VideoPage
├── utils/SandFilesHandle.ets        rawfile -> sandbox copy, delete, save-to-gallery
└── views
    ├── VideoPage.ets                the whole feature: overlay, gesture, presets, export
    └── XComponent.ets               the surface + player wiring
```

The documented tree matches the zip exactly.

**The design decision worth copying** is the split between `XComponent.ets` and
`VideoPage.ets`. `VideoXComponent` owns exactly one thing: getting a surface ID out
of `onLoad` and handing it to a fresh `AVPlayerDemo`. It knows nothing about
cropping. `VideoPage` stacks its own overlay on top of that component and never
touches the player. So the crop UI can be developed, restyled or reused without
going anywhere near the media state machine, and the player can be swapped for a
different source without touching the crop geometry.

**The decision worth avoiding** is in the same file: `VideoXComponent` starts
playback from `onLoad`, unconditionally, against a sandbox path that a separate
async copy may not have written yet (`HW-18-0048`). The two halves are decoupled in
structure and coupled in time, which is the worst of both.

## Implementation steps

1. **Copy the source video into the sandbox once**, in `aboutToAppear`, and only if
   it is not already there — `fs.accessSync(path + '/flower.mp4')`.
2. **Await that copy before starting playback** (`HW-18-0048`). `copyRawFileToSdcard`
   is a promise chain wrapping a callback API and returns nothing; make it return a
   promise and gate `onLoad` on it, or the first launch opens a zero-byte file.
3. **Size the frame from the display**, not from constants: `px2vp(display width)`
   for the width and `width / (16/9)` for the height, with the `Stack` carrying
   `.aspectRatio(StyleConstants.ASPECT_RATIO)` so the surface and the overlay cannot
   drift apart.
4. **Model the crop as four numbers** — `left`, `top`, `dynamicWidth`,
   `dynamicHeight` — and render it as the four border widths of an empty `Column`.
5. **Pick the edge by proximity.** In `onActionUpdate`, compare the finger's
   `localX`/`localY` against each edge and move only edges within `interval` (80 vp).
6. **Clamp in the same branch that moves.** Each of the four blocks has an if/else:
   the else applies the delta, the if pins the edge to the boundary and transfers
   the remainder into the opposite dimension.
7. **Debounce the gesture** at 50 ms so a drag does not re-render the border on
   every frame.
8. **Convert to source pixels with `vp2px` at the command boundary** — the frame is
   in vp, ffmpeg wants pixels.
9. **Delete the previous output with `unlink`, not `rmdir`** (`HW-18-0047`), or pass
   `-y` to ffmpeg, and re-enable the export button in the failure path.
10. **Save through `showAssetsCreationDialog`** — no album permission needed.
11. **Close the playback fd** on teardown; the sample leaves the `closeSync` commented
    out (`HW-18-0022`).
12. **Do not return a bare `true` from `onBackPress`** unless something else can leave
    the page (`HW-18-0049`).

## Verified snippets

All snippets are from `VideoCropping.zip`. Corrected forms are marked.

**The crop frame as a border — `entry/src/main/ets/views/VideoPage.ets`** (as shipped)

```typescript
Stack() {
  VideoXComponent();
  Column() {
  }.width(StyleConstants.FULL)
  .height(StyleConstants.FULL)
  .border({
    width: {
      left: this.left,
      top: this.top,
      right: this.videoWidth - this.left - this.dynamicWidth,
      bottom: this.videoHeight - this.top - this.dynamicHeight
    },
    color: $r('app.color.video_background_color'),
    style: BorderStyle.Solid
  })
  .gesture(
    PanGesture({ direction: PanDirection.Vertical | PanDirection.Horizontal })
      .onActionUpdate(this.debounce((e: GestureEvent) => { /* see below */ }, this.delay))
  );
}.width(this.videoWidth).aspectRatio(StyleConstants.ASPECT_RATIO)
.backgroundColor($r('app.color.dark_black'));
```

**Four border widths, one empty component.** The `Column` has no children and no
background; only its border paints. The left and top widths are the crop origin
directly, and the right and bottom are derived — `videoWidth - left - dynamicWidth`
— so the four numbers always describe one consistent rectangle and there is no way
to get a frame whose sides disagree. The border colour is a translucent dark, which
is what turns the border into a scrim rather than an outline.

`aspectRatio` on the `Stack` is load-bearing: it forces the overlay to have exactly
the same box as the video surface, so the vp coordinates the gesture produces map
onto the video without an offset term.

**The edge-picking gesture — same file** (as shipped)

```typescript
.onActionUpdate(this.debounce((e: GestureEvent) => {
  if (e.fingerList[0] === undefined || e.fingerList[0] === null) {
    return;
  }
  if (Math.abs(e.fingerList[0].localX - this.left) < this.interval) {      // near the left edge
    if (this.left + e.offsetX < 0) {
      this.dynamicWidth += this.left;
      this.left = 0;
    } else {
      this.left += e.offsetX;
      this.dynamicWidth -= e.offsetX;                                      // opposite edge stays put
    }
  }
  if (Math.abs(e.fingerList[0].localY - this.top) < this.interval) {       // near the top edge
    if (this.top + e.offsetY < 0) {
      this.dynamicHeight += this.top;
      this.top = 0;
    } else {
      this.top += e.offsetY;
      this.dynamicHeight -= e.offsetY;
    }
  }
  if (Math.abs(e.fingerList[0].localX - this.dynamicWidth - this.left) < this.interval) {
    if (this.left + this.dynamicWidth + e.offsetX > this.videoWidth) {
      this.dynamicWidth = this.videoWidth - this.left;                     // pin to the right bound
    } else {
      this.dynamicWidth += e.offsetX;
    }
  }
  if (Math.abs(e.fingerList[0].localY - this.dynamicHeight - this.top) < this.interval) {
    if (this.top + this.dynamicHeight + e.offsetY > this.videoHeight) {
      this.dynamicHeight = this.videoHeight - this.top;
    } else {
      this.dynamicHeight += e.offsetY;
    }
  }
}, this.delay))
```

**Moving a near edge must move the origin and the size together.** Dragging the
left edge right by 10 adds 10 to `left` *and* subtracts 10 from `dynamicWidth`, so
the right edge does not follow. Dragging the right edge only touches
`dynamicWidth`. That asymmetry is the whole difference between resizing and moving,
and it is why the near edges and far edges cannot share a code path.

Note the debounce wrapper is a method returning a closure, so `this.debounce(...)`
is evaluated once during `build()` and the returned function is what the gesture
holds — the `setTimeout`/`clearTimeout` pair inside it therefore shares one `timer`
across the whole drag, which is what makes it a debounce and not four independent
timers. The four `if`s are independent, not `else if`, which is deliberate: a finger
in a corner is near two edges and should move both.

`e.offsetX` in ArkUI's `PanGesture` is cumulative from the gesture start, not
per-frame — combined with the debounce that means a fast drag applies the total
offset in one step. It is visually acceptable here because the frame is a scrim,
but it is a real limitation if you need a pixel-accurate handle.

**Export: crop then hand to the gallery — same file** (corrected, see `HW-18-0047`)

```typescript
Button($r('app.string.export'))
  .enabled(this.disabled)
  .onClick(() => {
    this.disabled = false;                                  // button off while the job runs
    let copyContext = this.context;
    let pathDir = this.context.filesDir;
    if (fs.accessSync(pathDir + '/result/flowercropped.mp4')) {
      fs.unlinkSync(pathDir + '/result/flowercropped.mp4');  // FIX: sample calls deleteFile() -> rmdir()
    }
    let _that = this;
    let callBack: ICallBack = {
      callBackResult() {
        saveToFile(copyContext, '/result/flowercropped.mp4')
          .then(() => {
            _that.disabled = true;
          });
        hilog.info(DOMAIN, TAG, `successfully cropped`);
      }
    };
    MP4Parser.ffmpegCmd(`ffmpeg -y -i ${pathDir + '/flower.mp4'} -vf crop=${this.getUIContext()
      .vp2px(this.dynamicWidth)}:${this.getUIContext().vp2px(this.dynamicHeight)}:${this.getUIContext()
      .vp2px(this.left)}:${this.getUIContext().vp2px(this.top)} ${pathDir +
      '/result/flowercropped.mp4'}`, callBack);            // FIX: -y added; sample has no overwrite flag
  });
```

**`crop=w:h:x:y` takes source pixels, and the frame is in vp**, hence four
`vp2px` calls at the boundary. Doing the conversion here rather than storing pixels
in state is the right side of the line: the UI reasons in vp throughout, and only
the command string speaks the file's coordinate system.

`this.disabled` is a misnomer — it is the `enabled` flag, true meaning usable —
and it is set false before the job and true again only inside the success callback.
With no failure branch, one failed crop disables the button for the rest of the
session. That, plus `deleteFile` calling `fileIo.rmdir` on a regular file (which
fails `ENOTDIR` every time, leaving the stale output in place) and ffmpeg's default
refusal to overwrite, is `HW-18-0047`: the second export never happens and the
button never comes back.

`_that` captured for the callback is a plain-JS habit; `callBackResult` is a method
shorthand and could be an arrow function bound to `this` instead.

**Save to the gallery without a permission — `entry/src/main/ets/utils/SandFilesHandle.ets`** (as shipped)

```typescript
export async function saveToFile(context: Context, filename: string) {
  let uri: string = context.filesDir + filename;
  let phAccessHelper = photoAccessHelper.getPhotoAccessHelper(context);
  try {
    let srcFileUris: Array<string> = [fileUri.getUriFromPath(uri)];
    let photoCreationConfigs: Array<photoAccessHelper.PhotoCreationConfig> = [
      { fileNameExtension: 'mp4', photoType: photoAccessHelper.PhotoType.VIDEO }
    ];
    // 基于弹窗授权的方式获取媒体库的目标uri
    let desFileUris: Array<string> = await phAccessHelper.showAssetsCreationDialog(srcFileUris, photoCreationConfigs);
    let desFile: fileIo.File = await fileIo.open(desFileUris[0], fileIo.OpenMode.WRITE_ONLY);
    let srcFile: fileIo.File = await fileIo.open(uri, fileIo.OpenMode.READ_ONLY);
    await fileIo.copyFile(srcFile.fd, desFile.fd);
    fileIo.closeSync(srcFile);
    fileIo.closeSync(desFile);
    promptAction.showToast({ message: $r('app.string.save_success'), duration: 3000 });
  } catch (err) {
    hilog.error(DOMAIN, TAG, `failed to create asset by dialog errCode is: ${err.code}, ${err.message}`);
  }
}
```

**`showAssetsCreationDialog` is the permission-free way into the gallery.** The app
hands the system a list of sandbox URIs plus the metadata it wants for each; the
system shows a confirmation sheet and returns media-library URIs the app may write
to. No `WRITE_IMAGEVIDEO`, no runtime request. This sample gets it right and
declares no permissions at all — compare `PHOTO-18` and `PHOTO-20`, which use
equivalent permission-free flows and still declare `WRITE_IMAGEVIDEO` (`HW-18-0004`).

Two flaws remain: `desFileUris[0]` is indexed without checking whether the user
dismissed the sheet, and both `closeSync` calls sit outside a `finally`, so a
failed `copyFile` leaks both descriptors.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`, which is correct: the
picker is not used, the source is a bundled rawfile, and the export goes through
`showAssetsCreationDialog`.

The module declares `phone`, `tablet` and `2in1`, and `EntryAbility` writes
`topRectHeight`/`bottomRectHeight` into `AppStorage` for `MainPage` to consume via
`@StorageProp`. The dependency on `@ohos/mp4parser` (V2.0.7) is the only third-party
package, and it bundles ffmpeg — check its licence terms before shipping.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The ratio presets are hardcoded rectangles**, not computed ratios.
  `ratioParams` is a table of six `[left, top, width, height]` quadruples such as
  `[26, 15, 320, 180]` for 16:9, sized for one particular display width. On a tablet
  or a resized 2in1 window the presets no longer match the video box, and only the
  原图 branch — which recomputes from `videoWidth`/`videoHeight` — stays correct.
- `ASPECT_RATIO` is fixed at 16/9, so a portrait or square source video is letterboxed
  and the crop coordinates refer to the letterboxed box, not the source frame.
- `XComponent.setXComponentSurfaceRect` is called with the constants 1280x720; a
  source of any other resolution is scaled into that surface.
- The bottom function row (裁剪 / 比例 / 旋转 / 录制) is decoration — only 比例 is
  highlighted and none of the four has a click handler.
- The crop is not previewed against the ratio: dragging an edge after picking 16:9
  silently leaves that ratio.

## Pitfalls

- **`HW-18-0047` (B/medium, confirmed) — `deleteFile` calls `fileIo.rmdir` on a
  regular file.** Both callers pass `.../result/flowercropped.mp4`, so the delete
  fails `ENOTDIR` every time and the stale output survives. ffmpeg is invoked without
  `-y`, so it refuses to overwrite, `callBackResult` never fires, `saveToFile` never
  runs, and the export button — disabled at the start of the click and re-enabled only
  on success — stays dead for the session. Fix: `fileIo.unlink` for files (or add
  `-y`), and restore the flag in a failure path.
- **`HW-18-0048` (B/medium, probable) — first-launch race between the rawfile copy
  and playback.** `aboutToAppear` kicks off an async `mkdir` + `getRawFileContent`
  chain while `XComponent.onLoad` immediately calls `avPlayerUrlDemo(path)`, which
  opens with `CREATE | READ_WRITE` and so happily creates an empty file. The player
  errors, the error handler resets to `idle`, and the `idle` case calls `release()`
  — the player is gone for good and the video never plays until the app is
  restarted. Fix: await the copy, or gate `onLoad` playback on its promise.
- **`HW-18-0049` (B/low, confirmed) — the page cannot be left.** `MainPage.onBackPress`
  returns `true` unconditionally, vetoing the system back gesture, and the on-screen
  back arrow in `VideoPage` has no `onClick`. Fix: remove the blanket `true` or wire
  the arrow.
- **`HW-18-0022` (B/low, confirmed) — the playback fd is never closed.** Systematic
  across five photography samples. `AVPlayerDemo.avPlayerUrlDemo` opens the sandbox
  file, hands `fd://<n>` to the player, and leaves `// fs.closeSync(file);` commented
  out with no teardown path — one descriptor leaked per page entry for the process
  lifetime. Fix: close the file when the player is released, in `aboutToDisappear`.
- **Beyond the filed findings:** the `AVPlayerDemo` state machine calls `release()`
  from the `idle` case, which makes every recoverable error terminal, and
  `saveToFile` constructs a bare `new UIContext()` instead of taking the page's —
  the same anti-pattern appears in `VideoPage` and `XComponent`. Use
  `getUIContext()` from the component.

## References

- `huawei_industry_tree/18_photography/docs/17_video_cropping.md` — the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_cropping-0000002385796378
- `documentation/harmonyos-guides/02_media/video-playback.md` — the AVPlayer playback flow and state machine
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/video-playback
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` — `url`, `surfaceId`, `prepare`, `play`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` — `XComponentType.SURFACE`, `getXComponentSurfaceId`, `setXComponentSurfaceRect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` — `PanDirection`, `offsetX`/`offsetY` semantics, `fingerList`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-border.md` — per-side `border.width`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-border
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` — `showAssetsCreationDialog` and `PhotoCreationConfig`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `PHOTO-19` — the same mp4parser/ffmpeg export path, for a picture-in-picture merge
