---
id: PHOTO-25
title: Delayed video recording - a setTimeout countdown in front of AVRecorder.start, over a live preview
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/25_avrecorder_timer.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/avrecorder_timer-0000002529561491
sample: huawei_industry_tree/18_photography/downloads/AVRecorderTimer.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CameraKit", "@kit.CoreFileKit", "@kit.MediaKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit", "@kit.PreviewKit", "@kit.SensorServiceKit"]
apis: [UIContext, abilityAccessCtrl, camera, common, curves, dataSharePredicates, display, fileIo, filePreview, hilog, media, photoAccessHelper]
permissions: [ohos.permission.CAMERA, ohos.permission.MICROPHONE, ohos.permission.WRITE_IMAGEVIDEO, ohos.permission.READ_IMAGEVIDEO]
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0007, HW-18-0073, HW-18-0074, HW-18-0075, HW-18-0029, HW-18-0030, HW-18-0010, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when a camera feature needs to **start recording some seconds
after the user lets go of the button** - the self-timer on a video mode. The
pattern is small: build the whole camera-plus-recorder pipeline when the page
appears, then put a `setTimeout` and a visible countdown between the tap and
`avRecorder.start()`.

The transferable idea is the split between *prepared* and *started*. Everything
expensive - camera input, preview output, video output, session commit,
`AVRecorder.prepare` - happens at page load, so the deferred call is a single
`start()` that fires on time. Any "act in N seconds" control over an expensive
subsystem wants that shape: prepare eagerly, defer only the trigger.

The sample also carries the pieces that a real self-timer needs and that are
easy to forget: a `TextTimer` with a `ContentModifier` so the countdown renders
as a large number over the preview, a tap on the countdown that cancels it, and
`clearInterval` in `aboutToDisappear` so a pending timer cannot fire into a
destroyed page.

**Read `HW-18-0007` and `HW-18-0029` before adopting anything here.** The
shipped sample requests a permission it never declares, which fails the whole
grant, and closes the recording file descriptor immediately after `prepare()`.
Both defects sit directly on the main path.

## Feature checklist

- The preview fills a `XComponent` surface at a 16:9 ratio, sized from the
  current window rather than a constant.
- A clock icon opens a menu with off / 3s / 7s; the label under it changes to
  倒计时 (countdown), 3秒 or 7秒.
- Tapping record with a delay selected shows a large countdown number centred
  over the preview and hides the option and control panels.
- Tapping the countdown cancels it and returns to the idle state.
- When the countdown reaches zero, recording starts and a running `TextTimer`
  replaces the countdown.
- Tapping the recording button stops the recorder, rebuilds the preview, and
  fills the thumbnail from the gallery.
- Tapping the thumbnail opens the recorded video in the system Photos app.
- A flash menu (off / on) and a two-finger pinch-to-zoom act on the live
  session.
- Icons counter-rotate with the gravity sensor so they stay upright when the
  device is turned.

## Architecture

One `entry` module. The page owns all UI state; the camera pipeline lives in
module-scope variables inside one utility file.

```
entry/src/main/ets
├── constants/CameraConstants.ets   ratios and icon sizes; VIDEO_RATIO = 16/9
├── entryability/EntryAbility.ets   loads Index, publishes the window to AppStorage
├── entrybackupability/
├── pages/Index.ets                 @Entry, 471 lines: preview, countdown, panels, thumbnail
└── utils
    ├── CameraShooter.ets           21 lines - forwards to videoRecording(false, ..., 0, ...)
    ├── GravityUtil.ets             one-shot gravity read, used to set the recording rotation
    ├── PreviewUtil.ets             picks the preview profile and the surface rect from window size
    └── VideoRecorder.ets           the whole pipeline: session, AVRecorder, start/stop, zoom, flash
```

The documented tree matches the zip.

**The design decision worth copying** is `PreviewUtil`. Neither the surface
size nor the preview profile is a constant: `getPreviewSize()` derives the
surface rectangle from the live window rectangle and the target ratio, and
`getTargetPreviewProfile()` scans the camera's `previewProfiles`, keeps only
profiles at the target ratio, and picks the one closest to the window's short
edge. That is why the same page works folded, unfolded and in a resized window
- the geometry is computed, not hardcoded. Compare `PHOTO-13`, which pins the
surface to `display.getDefaultDisplaySync()` and stretches on a foldable.

**The structure worth avoiding** is `VideoRecorder.ets`'s module-scope
pipeline. `file`, `previewOutput`, `cameraInput`, `avRecorder`, `videoOutput`,
`videoSession` and `uri` are all module-level `let`s shared by every exported
function. It reads cleanly, but it is exactly what lets a stray `let` inside
`videoRecording` shadow the module `previewOutput` and silently disable its
release (`HW-18-0073`), and it leaves no place to null a released handle, so
the `if (videoSession)` guards in `stopRecordPreview` stay truthy on dead
objects (`HW-18-0010`). A class instance with nullable fields costs three extra
lines and makes both defects impossible.

## Implementation steps

1. **Declare CAMERA, MICROPHONE, READ_IMAGEVIDEO and WRITE_IMAGEVIDEO** in
   `module.json5` with `reason` and `usedScene`, and request **exactly that
   set** at runtime. One undeclared entry rejects the whole call
   (`HW-18-0007`).
2. **Branch on the result** of `requestPermissionsFromUser` before building the
   pipeline, and attach a `.catch` (`HW-18-0030`).
3. **Size the surface from the window**: `setXComponentSurfaceRotation({ lock: true })`
   then `setXComponentSurfaceRect(...)` in `XComponent.onLoad`, and take
   `getXComponentSurfaceId()` there - the id does not exist earlier.
4. **Pick the preview profile by ratio, then by proximity** to the window's
   short edge; pick the video profile from the preview profile's aspect so the
   two streams agree.
5. **Open the target file once and keep it open.** `url: fd://${file.fd}` is
   read by the recorder for the whole session; close it after `stop()`, once
   (`HW-18-0029`).
6. **Assign the module-level `previewOutput`,** do not redeclare it with `let`
   inside the builder (`HW-18-0073`).
7. **Render the countdown with a `TextTimer` plus a `ContentModifier`** so the
   remaining seconds can be drawn as one large glyph, and make the countdown
   itself tappable to cancel.
8. **Defer only `startRecord()`** behind `setTimeout(..., this.countDownTime)`,
   and keep the handle so `aboutToDisappear` can `clearInterval` it.
9. **Create the gallery asset when recording starts, not when the preview is
   built,** and pass the recorded uri to the thumbnail lookup (`HW-18-0074`).
10. **Await the teardown before every rebuild** - camera switch, fold change,
    post-record - and null the handles (`HW-18-0010`, `HW-18-0075`).

## Verified snippets

All snippets are from `AVRecorderTimer.zip`. Corrected forms are marked.

**The countdown and the deferred start — `entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
class CountDownTimerModifier implements ContentModifier<TextTimerConfiguration> {
  applyContent(): WrappedBuilder<[TextTimerConfiguration]> {
    return wrapBuilder(buildTextTimer);
  }
}

const ONE_THOUSAND = 1000;

@Builder
function buildTextTimer(config: TextTimerConfiguration) {
  Text(Math.ceil(config.count / ONE_THOUSAND - config.elapsedTime / 100).toString())
    .width(CameraConstants.FULL_PERCENT)
    .height(CameraConstants.FULL_PERCENT)
    .fontSize($r('app.float.count_down_font_size'))
    .fontColor(Color.White)
    .textAlign(TextAlign.Center)
}

// in build():
TextTimer({ isCountDown: true, count: this.countDownTime, controller: this.countDownTimeController })
  .contentModifier(this.countDownTimeModifier)
  .visibility(this.isCountingDown ? Visibility.Visible : Visibility.Hidden)
  .size(this.countDownSize)
  .offset(this.countDownOffset)
  .zIndex(1)
  .onClick(() => {
    clearInterval(this.countDownInterval);      // cancel the pending start
    this.isCountingDown = false;
    this.countDownTimeController.reset();
  })

Image($r('app.media.record'))
  .height(CameraConstants.CAPTURE_SIZE)
  .visibility(this.recording ? Visibility.Hidden : Visibility.Visible)
  .onClick(async () => {
    if (this.countDownTime !== 0) {
      this.isCountingDown = true;
      this.countDownTimeController.start();     // drives the visible number
    }
    this.countDownInterval = setTimeout(() => {
      this.isCountingDown = false;
      this.countDownTimeController.reset();
      startRecord();                            // the only deferred work
      this.textTimerController.start();         // the elapsed-time timer
      this.recording = true;
    }, this.countDownTime);
  })
```

**Two timers, one delay.** `TextTimer` with `isCountDown: true` draws the
number; `setTimeout` decides when recording actually begins. They are not wired
to each other, which is deliberate: the visual timer can be reset without
touching the pending start, and the pending start can be cancelled by
`clearInterval` without waiting for the animation. `countDownTime` is `0`, `3000`
or `7000` and doubles as the "no delay" flag - with `0`, `setTimeout` still runs
but on the next tick, so the button path is identical in both cases.

The `ContentModifier` is what makes the countdown a full-screen glyph rather
than a small label: `applyContent` returns a `WrappedBuilder`, and
`config.count` and `config.elapsedTime` give the builder the remaining
milliseconds. `.size(this.countDownSize)` and `.offset(this.countDownOffset)`
are recomputed by `updatePreview` so the number stays centred over the preview
rectangle rather than over the window.

**The permission request — same file** (corrected, see `HW-18-0007`, `HW-18-0030`)

```typescript
permissions: Array<Permissions> = [
  'ohos.permission.CAMERA',
  'ohos.permission.MICROPHONE',
  // FIX: 'ohos.permission.MEDIA_LOCATION' shipped here but is NOT in module.json5
  'ohos.permission.READ_IMAGEVIDEO',
  'ohos.permission.WRITE_IMAGEVIDEO'
];

async aboutToAppear() {
  try {
    this.updatePreview(getPreviewSize());
    const data = await abilityAccessCtrl.createAtManager()
      .requestPermissionsFromUser(this.context, this.permissions);
    // FIX: the sample ignores the result entirely and builds the pipeline regardless
    if (data.authResults.some((r: number) => r !== 0)) {
      console.error('Camera or microphone permission denied');
      return;
    }
    setTimeout(async () => {
      this.windowClass?.on('windowRectChange', () => { /* ... */ });
      zoomRatioRange = await cameraShooting(cameraPosition, surfaceId, this.context, this.updatePreview);
    }, 200);
  } catch (error) {
    console.error(`The updatePreview call failed. error: ${JSON.stringify(error)}`);
  }
}
```

**One undeclared permission fails the whole array.** `requestPermissionsFromUser`
does not skip an entry it cannot honour - it rejects the call, so CAMERA and
MICROPHONE are never granted either and the recorder cannot start at all. The
shipped list carries `ohos.permission.MEDIA_LOCATION`, which `module.json5`
never declares; the same copy-paste appears in `PipwindowRecorder`. MEDIA_LOCATION
is a restricted permission for EXIF geotags and this feature has no use for it,
so removing it is the fix, not declaring it.

The second correction is the `authResults` branch. The sample's `.then(() => ...)`
takes no parameter at all, so denying the dialog still runs the whole pipeline
build and produces raw camera errors instead of a refused-state UI
(`HW-18-0030`, six samples in this industry).

**Preparing the recorder — `entry/src/main/ets/utils/VideoRecorder.ets`** (corrected, see `HW-18-0029`, `HW-18-0073`)

```typescript
let file: fileIo.File | undefined;              // FIX: may be unassigned on an early return

let options: photoAccessHelper.CreateOptions = { title: Date.now().toString() };
let accessHelper: photoAccessHelper.PhotoAccessHelper = photoAccessHelper.getPhotoAccessHelper(context);
let videoUri: string = await accessHelper.createAsset(photoAccessHelper.PhotoType.VIDEO, 'mp4', options);
file = fileIo.openSync(videoUri, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
let aVRecorderConfig: media.AVRecorderConfig = {
  audioSourceType: media.AudioSourceType.AUDIO_SOURCE_TYPE_MIC,
  videoSourceType: media.VideoSourceType.VIDEO_SOURCE_TYPE_SURFACE_YUV,
  profile: aVRecorderProfile,
  url: `fd://${file.fd.toString()}`,            // the recorder writes here until stop()
  rotation: cameraPosition === 0 ? 90 : 270
};
uri = videoUri;
avRecorder = await media.createAVRecorder();
await avRecorder.prepare(aVRecorderConfig);
let videoSurfaceId: string | undefined = await avRecorder.getInputSurface();
// ...
previewOutput = cameraManager.createPreviewOutput(previewProfile, surfaceId);  // FIX: no `let`
videoSession.addOutput(previewOutput);
videoSession.addOutput(videoOutput);
await videoSession.commitConfig();
await videoSession.start();
// FIX: the sample has `finally { fileIo.closeSync(file); }` here - remove it entirely
```

```typescript
export async function stopRecord(): Promise<string> {
  try {
    await avRecorder.stop();
    await avRecorder.release();
    if (file !== undefined) {                   // FIX: guard, and close exactly once
      fileIo.closeSync(file);
      file = undefined;
    }
  } catch (error) {
    console.error(`The stopRecord call failed. error: ${JSON.stringify(error)}`);
  }
  return uri;
}
```

**The `fd` in the url is a live handle, not a name.** `prepare()` records the
descriptor; every frame the recorder writes afterwards goes through it. The
shipped `finally` closes it the moment the pipeline finishes building, long
before `startRecord()` runs, and `stopRecord` then closes the same number a
second time - which either writes nothing or, if the number has been reused,
writes into an unrelated file. The AVRecorder guide closes the file only after
`stop()`. The same defect ships in `VideoRecording` and `PipwindowRecorder`
(`HW-18-0029`).

The one-word fix on the preview output matters just as much. `let previewOutput`
inside `videoRecording` creates a local that shadows the module variable for
the rest of the function; the module variable stays `undefined`, so
`stopRecordPreview`'s `if (previewOutput) previewOutput.release()` never runs
and every rebuild leaks a preview stream (`HW-18-0073`).

**Teardown before rebuild — same file and `Index.ets`** (corrected, see `HW-18-0010`, `HW-18-0075`)

```typescript
export async function stopRecordPreview(): Promise<void> {
  try {
    if (videoSession) {
      await videoSession.stop();                // FIX: shipped code awaits none of these
    }
    if (cameraInput) {
      await cameraInput.close();
    }
    if (previewOutput) {
      await previewOutput.release();
      previewOutput = undefined;                // FIX: guards stay truthy on released objects
    }
    if (videoSession) {
      await videoSession.release();
      videoSession = undefined;
    }
    if (videoOutput) {
      await videoOutput.release();
      videoOutput = undefined;
    }
  } catch (error) {
    console.error(`The stopRecordPreview call failed. error: ${JSON.stringify(error)}`);
  }
}

// Index.ets, the fold-status branch of windowRectChange:
this.windowClass?.on('windowRectChange', async () => {
  const currFoldStatus = display.getFoldStatus();
  if (currFoldStatus !== this.foldStatus) {
    this.foldStatus = currFoldStatus;
    await stopRecordPreview();                  // FIX: absent in this branch only
    await videoRecording(false, cameraPosition, 0, surfaceId, this.context, this.updatePreview);
  } else {
    this.updatePreview(getPreviewSize());
  }
});
```

**Three rebuild paths, two of them tear down first.** After stopping a
recording and after a camera switch the sample calls `stopRecordPreview()` and
then `videoRecording(...)` - but neither is awaited, so the new session is
configured while the old input is still closing. The fold branch does not call
it at all, which on a foldable means a second session opening while the first
still holds the camera (`HW-18-0075`). Awaiting the teardown and nulling the
handles fixes both, and is the industry-wide pattern this sample is one of nine
instances of (`HW-18-0010`).

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.CAMERA",          "reason": "$string:reason_camera",
    "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.MICROPHONE",      "reason": "$string:reason_microphone",
    "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.WRITE_IMAGEVIDEO", "reason": "$string:reason_write_imagevideo",
    "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.READ_IMAGEVIDEO",  "reason": "$string:reason_read_imagevideo",
    "usedScene": { "abilities": ["EntryAbility"] } }
]
```

- All four are `user_grant`, so `reason` and `usedScene` are mandatory.
  READ_IMAGEVIDEO and WRITE_IMAGEVIDEO are **restricted** permissions and need
  an ACL entry in the signing profile.
- `usedScene` here omits `when`; the default is `inuse`, which is correct - the
  sample has no continuous task and must not hold the camera in the background.
- The runtime array in `Index.ets` must match this list exactly (`HW-18-0007`).
- `deviceTypes` is `["phone"]` only, although the code has an explicit
  fold-status branch.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- The recorder profile is fixed at 48 kHz AAC stereo and a 32 Mbit/s video
  bitrate, with `videoFrameWidth`/`Height` taken from the chosen video profile.
  `videoFrameRate` is 60 on the back camera and 30 on the front.
- The video profile search returns `undefined` when no profile matches the
  filter, and `AVRecorderProfile` is then built with `undefined` dimensions -
  there is no fallback.
- Zoom is clamped to a hardcoded upper bound of 15 rather than
  `zoomRatioRange[1]`; the lower bound does come from the device.
- The gravity subscription is registered in `aboutToAppear` and released in
  `aboutToDisappear`, but the rotation is forced to `0` whenever the display
  rotation is non-zero, so icon counter-rotation only works in portrait.
- `filePreview.closePreview` runs on every `onPageShow`, which is how returning
  from the system Photos app restores the camera page.

## Pitfalls

- **`HW-18-0007`** (D/high, confirmed): the runtime request array includes
  `ohos.permission.MEDIA_LOCATION`, which `module.json5` does not declare, so
  `requestPermissionsFromUser` rejects the whole call and CAMERA/MICROPHONE are
  never granted - the recorder cannot start. Same code in `PipwindowRecorder`.
  Fix: drop MEDIA_LOCATION from the request array.
- **`HW-18-0029`** (B/high, confirmed): the target `fd` is closed in a `finally`
  immediately after `prepare()`, before recording starts, and closed again in
  `stopRecord`. Recordings come out empty or corrupt. Fix: keep the file open
  until `stop()` completes and close it once, guarding for an unassigned file.
- **`HW-18-0073`** (B/medium, confirmed): `let previewOutput` inside
  `videoRecording` shadows the module variable, so `stopRecordPreview`'s
  release is dead code and preview streams leak on every rebuild. Fix: drop the
  `let`.
- **`HW-18-0074`** (B/medium, probable): `videoRecording` creates a gallery
  asset every time it runs, and `getThumbnail` queries the newest asset by
  `DATE_ADDED` - so the thumbnail and the preview point at the empty
  placeholder created by the rebuild, not at the recording. Fix: create the
  asset in `startRecord` and pass the recorded uri to `getThumbnail`.
- **`HW-18-0075`** (B/medium, probable): the fold-status branch of
  `windowRectChange` rebuilds the pipeline without calling `stopRecordPreview`
  first. Fix: `await stopRecordPreview()` before `videoRecording`.
- **`HW-18-0030`** (B/medium, confirmed): the permission result is ignored -
  `.then(() => ...)` takes no parameter and there is no `.catch`, so a denial
  still builds the camera pipeline. Six samples share this. Fix: branch on
  `authResults`.
- **`HW-18-0010`** (B/medium, confirmed): `stopRecordPreview` awaits nothing and
  nulls nothing, and its callers do not await it either, so a rebuild races the
  teardown and the truthiness guards fire on released handles. Nine samples in
  this industry. Fix: await each release, null the fields, await the helper.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-media-avrecorder.md` - `prepare`, `getInputSurface`, `start`, `stop`, and the fd lifetime
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avrecorder
- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` - `setTimeout` and `clearInterval`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-timer
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - `setXComponentSurfaceRect`, `setXComponentSurfaceRotation`, `getXComponentSurfaceId`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-references/04_media/arkts-apis-camera-previewoutput.md` - preview output creation and release
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-camera-previewoutput
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - `requestPermissionsFromUser` and `authResults`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-guides/05_media/camera-recording-case.md` - the reference recording pipeline, including when to close the file
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-recording-case
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - CAMERA and MICROPHONE
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - READ_IMAGEVIDEO, WRITE_IMAGEVIDEO, MEDIA_LOCATION
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `PHOTO-06` - the same recorder pipeline without the timer, and the origin of `HW-18-0029`
- `PHOTO-30` - PipwindowRecorder, which carries the identical permission and fd defects
