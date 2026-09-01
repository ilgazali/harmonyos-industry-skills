---
id: PHOTO-15
title: Video trim timeline - drag two handles over a thumbnail strip, preview with fetchFrameByTime, cut with ffmpeg
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/15_video_clip.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_clip-0000002405051977
sample: huawei_industry_tree/18_photography/downloads/VideoClip.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [common, dataSharePredicates, display, fileIo, fileUri, hilog, image, media, photoAccessHelper, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0043, HW-18-0044, HW-18-0045, HW-18-0022, HW-18-0091, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when a screen must let the user **choose a sub-range of a video by
dragging** - the two-handle trim bar that every video editor, story composer and
clip-sharing flow has. The same construction serves audio trimming, GIF range
selection and any "pick a window inside a duration" control.

The transferable structure is a **one-way mapping between pixels and
milliseconds**, computed once. The strip is four thumbnails wide; that width
stands for the full duration; every handle position converts to a time by the
same ratio, and the minimum selectable range is expressed as the pixel width of
one second. Once those two numbers (`thumbnailWidth`, `oneSecondWidth`) exist,
every gesture handler is three lines of arithmetic and the clamping falls out of
the same expressions.

The cut itself is **not** a platform API. HarmonyOS has no ArkTS trimming
primitive, so the sample shells out to the OpenHarmony `@ohos/mp4parser`
package, which embeds ffmpeg and takes a command string. That is worth knowing
before you plan a feature around it: the dependency is third-party, the call is
callback-based, and the failure mode is an exit code. **Read `HW-18-0043` first**
- the sample's loading overlay never comes down if that exit code is non-zero.

## Feature checklist

- The home page lists demo video cards and a banner that opens the photo picker
  filtered to `VIDEO_TYPE`.
- Choosing a video pushes an edit page carrying the URI.
- The edit page shows the video, a play/pause control, the current time over the
  total, and a four-thumbnail strip covering the whole duration.
- Left and right handles drag along the strip; the region outside the selection
  is greyed out.
- Dragging either handle scrubs the player to that instant and, before first
  playback, swaps the preview frame for the frame at that time.
- The right handle cannot cross closer than one second to the left handle, and
  neither can leave the strip.
- A third drag target - the selected region itself - moves the playhead without
  changing the selection.
- Playback starts at the selected start time and pauses at the selected end.
- "保存" shows a full-screen loading overlay, cuts the range with ffmpeg, and
  offers the result to the gallery through the system save dialog.
- The three other edit tabs and the toolbar icons raise a "demo only" toast.

## Architecture

One `entry` module. The edit page is by far the largest file - 533 lines, all of
the gesture state, and the whole timeline built out of stacked `Row`s.

```
entry/src/main/ets
├── component/VideoForm.ets        the home grid's video card (cover, title, size/duration)
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model
│   ├── VideoParams.ets            IVideoFormData + five static demo entries
│   └── VideoSizeData.ets          photoSize (w/h) + totalTime, filled from metadata
├── pages
│   ├── MainPage.ets               @Entry, Navigation host, the video picker
│   └── VideoClipPage.ets          NavDestination: the whole trim UI (533 lines)
└── utils
    ├── Logger.ets
    └── VideoUtils.ets             duration query, ffmpeg clip, gallery save, formatting
```

The documented tree matches the zip.

**The design decision worth copying** is the layered `Stack` that makes up the
timeline. There is no custom drawing and no canvas: the same
`thumbnailShow()` builder is rendered three times at different widths and
opacities, and the handles are `Image`s positioned by `offset`.

```
Stack (width 80%, alignContent TopStart)
 ├─ thumbnails @ opacity 0.1, full width          the "outside the selection" ghost
 ├─ thumbnails clipped to rightWidth              the selected region, carries the scrub PanGesture
 ├─ grey Row clipped to leftWidth                 masks the part before the start handle
 ├─ thumbnails @ opacity 0.1 clipped to leftWidth re-ghosts that masked part
 ├─ Image(left_button)  .offset({x: leftWidth - 24})
 ├─ Image(right_button) .offset({x: rightWidth})
 └─ 2 vp blue Row       .offset({x: currentOffset})   the playhead
```

Because each layer is clipped to a state-bound width, moving a handle is a single
number assignment and ArkUI does the rest. The cost is that the strip is
rendered four times over, which for four `Image`s of one `PixelMap` each is
cheap - but it would not scale to a strip of twenty thumbnails.

**The decision worth avoiding** is `canDrag`. Every gesture handler opens with
`this.canDrag = false` and closes with `this.canDrag = true`, wrapping an
`await` in between - a hand-rolled re-entrancy lock over an async frame fetch.
It works, but it means a fast drag silently drops updates. Debouncing the
`fetchFrameByTime` call and letting the geometry update on every event would be
both smoother and simpler.

## Implementation steps

1. **Derive the geometry once in `aboutToAppear`:** `thumbnailWidth` is a fifth
   of the display width, the strip is four of those, and `rightWidth` starts at
   the full strip.
2. **Read the duration from the media library** (`getAssets` with
   `PhotoKeys.DURATION`) rather than from the player, so the timeline exists
   before the first frame decodes.
3. **Compute `oneSecondWidth = stripWidth / fullTime * 1000`** and use it as the
   minimum gap between the handles - it is the only place a "minimum clip
   length" rule needs to be expressed.
4. **Open the video once with `fileIo.openSync` and hand the fd to
   `AVImageGenerator.fdSrc`;** close that file in `aboutToDisappear` alongside
   the generator release (`HW-18-0022`).
5. **Fetch the four strip thumbnails at 0, 1/3, 2/3 and the end** with
   `fetchFrameByTime(..., AV_IMAGE_QUERY_CLOSEST_SYNC, videoSize.photoSize)`.
6. **Clamp inside the gesture condition, not after it.** Each `onActionUpdate`
   tests the prospective position against its neighbour before assigning, so an
   out-of-range drag is simply ignored.
7. **Persist the origin in `onActionEnd` and `onActionCancel`** - both, or a
   cancelled drag leaves the next one starting from a stale offset.
8. **Pause the player and `setCurrentTime` on every handle move,** and refresh
   the preview frame only while `isPlayed` is false.
9. **Pause playback at the real end time,** not a second early (`HW-18-0045`).
10. **Wrap the save in `try/finally` and clear `isClipping` there**
    (`HW-18-0043`), and guard each `closeSync` in `clipAndSaveVideo`'s `finally`
    (`HW-18-0044`).

## Verified snippets

All snippets are from `VideoClip.zip`. Corrected forms are marked.

**The geometry and the right handle — `entry/src/main/ets/pages/VideoClipPage.ets`** (as shipped)

```typescript
aboutToAppear(): void {
  this.thumbnailWidth = this.getUIContext().px2vp(display.getDefaultDisplaySync().width) / 5;
  this.rightWidth = this.thumbnailWidth * 4;
  this.rightWidthOrigin = this.rightWidth;
}

async initPix() {
  this.fullTime = await VideoUtils.getVideoDuration(this.selectUri, this.getUIContext());
  this.oneSecondWidth = this.thumbnailWidth * 4 / this.fullTime * 1000;   // px per second
  this.endTime = this.fullTime;
  // ...
}

Image($r('app.media.right_button'))
  .height(THUMBNAIL_HEIGHT)
  .width(24)
  .offset({ x: this.rightWidth })
  .gesture(
    PanGesture({ fingers: 1 })
      .onActionUpdate(async event => {
        if (this.rightWidthOrigin + event.offsetX <= this.thumbnailWidth * 4 &&
          this.rightWidthOrigin + event.offsetX >= this.leftWidth + this.oneSecondWidth &&
        this.canDrag) {
          this.canDrag = false;
          this.rightWidth = this.rightWidthOrigin + event.offsetX;
          this.endTime = (this.rightWidth / this.thumbnailWidth) / 4 * this.fullTime;
          this.currentTime = this.endTime;
          this.videoController.pause();
          this.isVideoStart = false;
          this.videoController.setCurrentTime(this.currentTime / 1000);
          this.currentTimeShow = VideoUtils.formatDuration(this.currentTime);
          if (!this.isPlayed) {
            this.pixelMapShow = await this.fetchFrameByTime(this.currentTime * 1000);
          }
          this.canDrag = true;
        }
      })
      .onActionEnd(() => {
        this.rightWidthOrigin = this.rightWidth;
        this.currentOffsetOrigin = this.currentOffset;
      })
  );
```

**`event.offsetX` is cumulative for the whole gesture, not incremental**, which
is why every handler adds it to an `*Origin` field captured at the end of the
previous drag rather than to the live value. Forgetting that is the classic
`PanGesture` bug: adding a cumulative offset to a live position accelerates the
drag quadratically.

The two-part condition is the entire clamping logic. The upper bound keeps the
handle on the strip; the lower bound, `leftWidth + oneSecondWidth`, enforces a
one-second minimum selection in pixel space. Expressing the rule in pixels
rather than milliseconds means the same comparison both validates the drag and
positions the handle - there is no separate "convert, check, convert back".

Note `currentTime` is in **milliseconds** throughout, while
`VideoController.setCurrentTime` takes **seconds** - hence the `/ 1000` at every
call site, and `* 1000` going into `fetchFrameByTime`, which takes microseconds.
Three time units in six lines; label them or you will lose an hour.

**The frame source, with the file closed — same file** (corrected, see `HW-18-0022`)

```typescript
async imageGeneratorGetThumbnail() {
  if (this.selectUri === '') {
    return;
  }
  if (this.avImageGenerator) {
    await this.avImageGenerator.release();
  }
  this.fileAlbum = fileIo.openSync(this.selectUri, fileIo.OpenMode.READ_ONLY);
  this.avFileDescriptor = { fd: this.fileAlbum.fd };
  this.videoSize = await VideoUtils.getVideoData(this.avFileDescriptor);
  this.avImageGenerator = await media.createAVImageGenerator();
  this.avImageGenerator.fdSrc = this.avFileDescriptor;
}

async fetchFrameByTime(time: number): Promise<image.PixelMap | undefined> {
  let pixelMap = await this.avImageGenerator?.fetchFrameByTime(time,
    media.AVImageQueryOptions.AV_IMAGE_QUERY_CLOSEST_SYNC, this.videoSize.photoSize);
  return pixelMap;
}

aboutToDisappear(): void {
  this.avImageGenerator?.release();
  if (this.fileAlbum) {                       // FIX: the sample never closes the opened file
    fileIo.closeSync(this.fileAlbum);
    this.fileAlbum = undefined;
  }
}
```

**One open file backs both the metadata read and every thumbnail.**
`getVideoData` builds an `AVMetadataExtractor` on the same descriptor to learn
the frame size, which is then passed to `fetchFrameByTime` as `PixelMapParams` -
so the decoded frames come back at the video's native resolution rather than a
default. That is why the strip images look sharp and why they are also the
largest allocations on the page.

`AV_IMAGE_QUERY_CLOSEST_SYNC` is the deliberate choice for a scrubber: it
returns the frame nearest the requested time whether or not it is a keyframe,
which is what a user dragging a handle expects. The keyframe-only variants are
faster but make the preview jump.

The fd itself is leaked in the shipped code: `fileAlbum` is opened on every
visit to the page and never closed, one descriptor per edit session for the
lifetime of the process. `aboutToDisappear` already releases the generator, so
closing the file there costs two lines. Five photography samples share this
defect (`HW-18-0022`).

**The clip and the save — `entry/src/main/ets/utils/VideoUtils.ets` and the save
button** (corrected, see `HW-18-0043`, `HW-18-0044`)

```typescript
// VideoClipPage.ets - the Save button
Button($r('app.string.save'))
  .onClick(async () => {
    this.isClipping = true;
    let sTime = (Math.floor(this.startTime / 1000)).toString();
    let eTime = (Math.floor(this.endTime / 1000) - Math.floor(this.startTime / 1000)).toString();
    try {
      await VideoUtils.clipAndSaveVideo(sTime, eTime, this.selectUri, this.getUIContext());
    } catch (err) {                                   // FIX: the sample has neither try nor catch
      Logger.error(`clip failed: ${JSON.stringify(err)}`);
      // and toast a failure string - the sample's resources have save_success but no failure message
    } finally {
      this.isClipping = false;                        // FIX: was only reset on success
    }
  });

// VideoUtils.ets - the cut
async clipVideo(sTime: string, eTime: string, selectedVideoUri: string, context: UIContext): Promise<boolean> {
  await this.copyVideo(selectedVideoUri, context);
  let videoDir = context.getHostContext()?.cacheDir + '/videoDir';
  let isSuccess = false;
  await new Promise<void>((resolve, reject) => {
    MP4Parser.ffmpegCmd(`ffmpeg -y -ss ${sTime} -i ${videoDir}/InputVideo.mp4 -t ${eTime} -c copy -avoid_negative_ts make_zero ${videoDir}/OutputVideo.mp4`,
      {
        callBackResult: (code: number) => {
          if (code === 0) {
            isSuccess = true;
            resolve();
          } else {
            isSuccess = false;
            reject(new Error(`Clip video error,error code: ${code}`));
          }
        },
      }
    );
  });
  return isSuccess;
}

// VideoUtils.ets - copying the result into the asset the dialog created
let desFile: fs.File | undefined = undefined;
let srcFile: fs.File | undefined = undefined;
try {
  desFile = await fs.open(desFileUris[0], fs.OpenMode.WRITE_ONLY);
  srcFile = await fs.open(srcFileUri, fs.OpenMode.READ_ONLY);
  await fs.copyFile(srcFile.fd, desFile.fd);
} finally {
  if (srcFile) {                                     // FIX: shipped finally closes undefined files
    fs.closeSync(srcFile);
  }
  if (desFile) {
    fs.closeSync(desFile);
  }
}
```

**`-ss` before `-i`, `-t` as a duration, `-c copy`.** Those three flags are the
design. Putting `-ss` ahead of the input makes ffmpeg seek before decoding
rather than reading and discarding, which is what makes a trim near the end of a
long video fast. `-t` is a **length**, not an end timestamp - hence the
subtraction when building `eTime`. `-c copy` streams the packets through without
re-encoding, so the cut is near-instant and lossless, at the price of snapping
the real start to the nearest keyframe; `-avoid_negative_ts make_zero` rebases
the timestamps so players do not choke on the result.

Two defects meet in this flow. The promise rejects on a non-zero exit code, and
neither `clipAndSaveVideo` nor the button awaits it inside a `try`, so a failed
cut leaves `isClipping` true forever - the full-screen `LoadingProgress` overlay
stays up and the page is unusable until the app is killed (`HW-18-0043`). And if
the user declines the `showAssetsCreationDialog`, `desFileUris` is empty,
`fs.open(undefined)` throws, and the un-guarded `finally` calls
`fs.closeSync(undefined)` - which throws a `TypeError` that replaces the real
error, and leaks the one descriptor that did open (`HW-18-0044`).

Note also that `clipVideo` copies the whole source video into
`cacheDir/videoDir` first, deleting and recreating the directory each time. For
a long video that is a full-size copy before any trimming happens.

**Pausing at the selected end — `entry/src/main/ets/pages/VideoClipPage.ets`**
(corrected, see `HW-18-0045`)

```typescript
Video({ src: this.selectUri, controller: this.videoController, previewUri: this.pixelMapShow })
  .controls(false)
  .onStart(() => {
    this.isPlayed = true;
    this.isVideoStart = true;
    this.videoController.setCurrentTime(this.startTime / 1000);   // playback begins at the selection
    this.currentOffset = this.startTime / this.fullTime * this.thumbnailWidth * 4;
    this.currentOffsetOrigin = this.currentOffset;
  })
  .onUpdate((event) => {
    if (event) {
      this.currentTimeShow = VideoUtils.formatDurationSecond(event.time);
      if (event.time * 1000 / this.fullTime * this.thumbnailWidth * 4 >= this.currentOffsetOrigin) {
        this.currentOffset = event.time * 1000 / this.fullTime * this.thumbnailWidth * 4;
      }
      // Video play to the end position
      if (event.time >= this.endTime / 1000) {        // FIX: shipped code subtracts a whole second
        this.videoController.pause();
        this.isVideoStart = false;
        this.currentOffset = this.endTime / this.fullTime * this.thumbnailWidth * 4;
        this.currentOffsetOrigin = this.currentOffset;
      }
    }
  });
```

**`onUpdate` is the playhead's only driver**, and `event.time` is in seconds
while `endTime` is in milliseconds - the `/ 1000` is the unit conversion, and the
`- 1` the sample adds to it is a whole second of the user's selection thrown
away. With a ten-second selection the preview stops at nine, so what the user
watches is never what gets exported.

The `>=` guard on `currentOffsetOrigin` is subtler and correct: it stops the
playhead from jumping backwards when `onUpdate` fires with a stale time
immediately after a seek.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. Reading the source
video goes through `PhotoViewPicker`, and writing the result goes through
`showAssetsCreationDialog`, which creates the gallery asset on the user's
confirmation - both grant per-file access without a media permission.

The third-party dependency is the thing to plan for: `@ohos/mp4parser` from
ohpm, imported as `import { MP4Parser } from '@ohos/mp4parser'`. It ships an
ffmpeg build, so it is a meaningful addition to the package size and its
licensing needs checking before shipping.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The strip is exactly four thumbnails and the geometry is derived from
  `display.getDefaultDisplaySync().width` in `aboutToAppear`, so the timeline
  does not respond to a window resize on 2in1 or to a fold/unfold.
- `-c copy` means cuts land on keyframes; frame-accurate trimming would require
  re-encoding (drop `-c copy`), which is far slower.
- `sTime` and `eTime` are floored to whole **seconds**, so sub-second precision
  in the handles is discarded at export.
- The cut always writes `videoDir/OutputVideo.mp4` and `initializeFolder` wipes
  the directory each run - there is no concurrent-clip support.
- Only the first tab of each `Tabs` has content; `onContentWillChange` returns
  `false` for the others by design.

## Pitfalls

- **`HW-18-0043`** (B/medium, confirmed): a clip failure leaves `isClipping`
  true. The Save handler sets the flag, awaits `clipAndSaveVideo` with no
  try/catch, and clears it only on the success path, while `clipVideo` rejects on
  any non-zero ffmpeg exit code - so the modal `LoadingProgress` overlay locks
  the UI permanently, plus an unhandled rejection. Fix: `try/finally`, reset in
  `finally`, toast on error.
- **`HW-18-0044`** (B/low, confirmed): the save-dialog refusal path.
  `desFile`/`srcFile` start `undefined`; declining `showAssetsCreationDialog`
  makes `fs.open(desFileUris[0])` throw, and the `finally` then calls
  `fs.closeSync(undefined)`, masking the real error and leaking the descriptor
  that did open. Fix: guard each close.
- **`HW-18-0045`** (B/low, probable): the preview pauses a full second before the
  selected end (`event.time >= this.endTime / 1000 - 1`), so the last second of
  every selection is never previewed even though it is exported. Fix: drop the
  `- 1`.
- **`HW-18-0022`** (B/low, confirmed): systematic - `fileIo.openSync` results are
  never closed. Here `fileAlbum` is opened per page visit and leaked; the same
  defect appears in Picture, FaceDetection, CropRect and VideoCropping. Fix:
  close it in `aboutToDisappear`.
- **`getVideoDuration` assumes the URI is in the media library.** It queries
  `getAssets` with an `equalTo('uri', ...)` predicate and calls
  `getFirstObject()` with no empty check, so a URI from anywhere else rejects
  inside `initPix` and the page loads with `fullTime === 0` - which makes every
  pixel-to-time ratio a division by zero.
- **`canDrag` drops gesture events.** The flag is cleared for the duration of an
  awaited `fetchFrameByTime`, so frames arriving during a fast drag are skipped
  rather than queued; the handle visibly lags the finger on long videos.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `PanGesture`, cumulative `offsetX`, `onActionEnd` / `onActionCancel`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/04_media/arkts-apis-media-avimagegenerator.md` - `fetchFrameByTime`, `AVImageQueryOptions`, `PixelMapParams`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avimagegenerator
- `documentation/harmonyos-references/04_media/arkts-apis-media-avmetadataextractor.md` - `fetchMetadata`, `videoWidth` / `videoHeight` / `duration`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avmetadataextractor
- `documentation/harmonyos-references/02_application-framework/ts-media-components-video.md` - `VideoController`, `onStart`, `onUpdate`, `previewUri`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-media-components-video
- `documentation/harmonyos-references/04_media/arkts-apis-photoAccessHelper-PhotoAccessHelper.md` - `getAssets`, `PhotoKeys.DURATION`, `showAssetsCreationDialog`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-guides/05_media/photoaccesshelper-photoviewpicker.md` - `PhotoViewPicker` with `VIDEO_TYPE`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `open`, `copyFile`, `closeSync`, `copy`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `PHOTO-17` (VideoCropping) - the spatial counterpart: cropping the frame rather than the duration
