---
id: PHOTO-30
title: Keep recording in a picture-in-picture window - auto-enter PiP when the camera app leaves the foreground
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/30_pipwindow_recorder.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/pipwindow_recorder-0000002522526388
sample: huawei_industry_tree/18_photography/downloads/PipwindowRecorder.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CameraKit", "@kit.CoreFileKit", "@kit.MediaKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit", "@kit.SensorServiceKit"]
apis: [UIContext, abilityAccessCtrl, camera, common, curves, dataSharePredicates, display, fileIo, hilog, media, photoAccessHelper, sensor, PiPWindow]
permissions: [ohos.permission.CAMERA, ohos.permission.MICROPHONE, ohos.permission.WRITE_IMAGEVIDEO, ohos.permission.READ_IMAGEVIDEO]
min_api: 20
modules: [entry]
findings: [HW-18-0088, HW-18-0007, HW-18-0010, HW-18-0023, HW-18-0029, HW-18-0030, HW-18-0041, HW-18-0073, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when **a live capture must survive the app going to the
background**. A video recorder that stops the moment the user checks a message
is not a recorder; the platform answer is a picture-in-picture window that
inherits the same `XComponent` surface, so the camera session and the
`AVRecorder` never learn that the app left the foreground.

The mechanism is small: create a `PiPWindow.PiPController` bound to the
preview's `XComponentController`, call `setAutoStartEnabled(true)`, and the
window system does the transition for you on every background event. The page
keeps owning the camera; only the surface changes host.

It generalises to any long-running foreground capture or playback: a video
call, a live-stream preview, a navigation camera, a media player. **Two things
must be read before adopting it.** The sample's runtime permission request is
broken in a way that stops the feature dead (`HW-18-0007`), and the document
promises a stop-and-save on PiP close that the code does not implement
(`HW-18-0088`). Both are one-line fixes and both are load-bearing.

## Feature checklist

- The page opens on a live camera preview inside a fixed-ratio `XComponent`
  with the surface rotation locked; recording is only offered once CAMERA and
  MICROPHONE are granted.
- Pressing the record button starts an `AVRecorder` whose rotation is derived
  from the gravity sensor at the moment recording starts.
- Sending the app to the background raises a PiP window that keeps showing the
  live preview, and recording continues.
- Returning to the app restores the preview to the page.
- Closing the PiP window stops the recording and finalises the video into the
  gallery.
- Pressing stop in the app saves the video, rebuilds the preview session, and
  shows the new video's thumbnail in the corner.
- Pinch-to-zoom scales the preview between the session's real zoom bounds.

## Architecture

One `entry` module. The page is a component; all camera and recorder state
lives in module-level variables inside `VideoRecorder.ets`.

```
entry/src/main/ets
├── constants/CameraConstants.ets    ratios and control sizes
├── entryability/EntryAbility.ets    full screen; publishes the window into AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages/Index.ets                  @Entry: PiP controller, XComponent, control panel, thumbnail
└── utils
    ├── CameraShooter.ets            the photo path (unused by this scenario)
    ├── GravityUtil.ets              one-shot gravity read -> device degree
    ├── PreviewUtil.ets              window/display sizes -> preview size and preview profile
    └── VideoRecorder.ets            camera session + AVRecorder: build, start, stop, zoom
```

The documented tree matches the zip.

**The design decision worth copying** is `PreviewUtil.getTargetPreviewProfile`.
Every other camera sample in this industry hardcodes a profile (`1440x1080`)
and looks it up with `find()`, which returns `undefined` on any device that
does not expose exactly that size (`HW-18-0021`). This one instead walks the
whole `previewProfiles` array, keeps only profiles whose aspect ratio equals
the target, and among those picks the one closest to the current window size.
The result degrades on unknown hardware instead of failing. If you copy one
file out of this sample, copy that one.

**The decision worth avoiding** is the module-level mutable state in
`VideoRecorder.ets` - `let previewOutput`, `let cameraInput`, `let avRecorder`,
`let videoSession`, `let file`, `let uri`. It is what lets `videoRecording()`
shadow the module `previewOutput` with a local of the same name
(`HW-18-0073`), so the preview output is never released, and it is why
`Index.ets` keeps a second set of module globals (`surfaceId`,
`zoomRatioRange`, `cameraPosition`) that drift from the first. A class with
fields, instantiated once, removes both failure modes.

## Implementation steps

1. **Declare exactly the permissions you request.** `requestPermissionsFromUser`
   rejects the entire call if any name in the array is not in `module.json5`
   (`HW-18-0007`).
2. **Read `authResults` before building anything.** The sample's `.then()`
   takes no parameter and starts the recorder pipeline unconditionally
   (`HW-18-0030`).
3. **Take `surfaceId` in `XComponent.onLoad`, after
   `setXComponentSurfaceRotation({ lock: true })` and
   `setXComponentSurfaceRect(...)`,** and start the camera from there - not
   from a 200 ms `setTimeout` (`HW-18-0041`).
4. **Create the PiP controller with the page's own `XComponentController`.**
   That shared controller is what lets the PiP window show the live camera
   surface rather than a copy.
5. **Call `setAutoStartEnabled(true)`** so the transition is driven by the
   window system on background, with no ability-lifecycle code of your own.
6. **Handle `PiPState.STOPPED` as a real event:** stop the recorder and
   finalise the asset there (`HW-18-0088`).
7. **Open the recording file once and close it once, after `avRecorder.stop()`**
   (`HW-18-0029`).
8. **Assign the returned `zoomRatioRange`** at every `videoRecording()` call
   site, or the pinch clamp compares against `undefined` (`HW-18-0023`).
9. **Release the preview output you actually created** - do not shadow the
   variable the teardown reads (`HW-18-0073`).
10. **Await the teardown before the rebuild** on the stop path (`HW-18-0010`),
    and **detach both PiP listeners in `aboutToDisappear`** - `off('stateChange')`,
    `off('controlPanelActionEvent')`, then drop the controller. The sample does
    the second part correctly.

## Verified snippets

All snippets are from `PipwindowRecorder.zip`. Corrected forms are marked.

**The permission bootstrap — `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-18-0007`, `HW-18-0030`, `HW-18-0041`)

```typescript
permissions: Array<Permissions> = [
  'ohos.permission.CAMERA',
  'ohos.permission.MICROPHONE',
  // FIX: 'ohos.permission.MEDIA_LOCATION' removed - never declared in module.json5
  'ohos.permission.READ_IMAGEVIDEO',
  'ohos.permission.WRITE_IMAGEVIDEO'
];

async aboutToAppear() {
  try {
    const result = await abilityAccessCtrl.createAtManager()
      .requestPermissionsFromUser(this.context, this.permissions);
    if (result.authResults.some((r: number) => r !== 0)) {   // FIX: sample ignores the result
      hilog.error(DOMAIN, TAG, `Permissions denied: ${JSON.stringify(result.authResults)}`);
      return;
    }
    this.granted = true;                 // FIX: surface owner starts the camera, not a 200 ms timer
    this.startPip();
  } catch (error) {
    hilog.error(DOMAIN, TAG, `The updatePreview call failed. error: ${JSON.stringify(error)}`);
  }
}
```

**`ohos.permission.MEDIA_LOCATION` is the highest-severity defect in this
sample and it is invisible in a log.** `requestPermissionsFromUser` does not
partially fulfil a request: if a single name in the array is missing from
`module.json5`, the whole call is rejected. The sample's array has five
entries; `module.json5` declares four. So CAMERA and MICROPHONE are never
granted, the pipeline that the `.then` fires anyway hits raw permission errors,
and the feature the document demonstrates cannot start at all (`HW-18-0007` -
the identical array appears in `AVRecorderTimer`, `PHOTO-25`). MEDIA_LOCATION
is also a restricted permission with no role in this scenario: nothing here
reads EXIF GPS.

The shipped code compounds it. `.then(() => ...)` takes no argument, so a
denial is indistinguishable from a grant (`HW-18-0030`), and the pipeline is
started from `setTimeout(..., 200)` - a guess that the `XComponent` will have
produced its surface id by then (`HW-18-0041`). Both go away when the grant
gates a flag and `onLoad` owns the start.

**The PiP controller — same file** (as shipped)

```typescript
async createPipController() {
  try {
    this.pipController = await PiPWindow.create({
      context: this.context,
      componentController: this.mXComponentController,   // the page's own surface controller
      navigationId: this.navigationId,
      templateType: PiPWindow.PiPTemplateType.VIDEO_PLAY,
      controlGroups: [PiPWindow.VideoPlayControlGroup.VIDEO_PREVIOUS_NEXT]
    });

    this.pipController.setAutoStartEnabled(true);
    this.pipController.on('stateChange', (state: PiPWindow.PiPState, reason: string) => {
      this.onStateChange(state, reason);
    });
  } catch (e) {
    hilog.error(DOMAIN, TAG, `Failed to create pip controller. Cause:${e.code}, message:${e.message}`);
  }
}

async startPip() {
  if (!this.pipController) {
    await this.createPipController();
  }
  if (!this.pipController) {
    hilog.error(DOMAIN, TAG, `pipController create error`);
    return;
  }
  try {
    await this.pipController.startPiP();
  } catch (err) {
    hilog.error(DOMAIN, TAG, `StartPiP failed. Cause code: ${err.code}, message: ${err.message}`);
  }
}
```

**Three fields carry the design.** `componentController` is the whole trick:
handing PiP the same `XComponentController` the page renders with means the PiP
window displays the identical surface, so the camera session and the recorder
never see a teardown - the app backgrounds and the pixels simply keep going to
another window. `setAutoStartEnabled(true)` moves the trigger into the window
system; there is no `onBackground` handler in this project, and there does not
need to be. `templateType: VIDEO_PLAY` selects the control layout the floating
window draws.

`controlGroups: [VIDEO_PREVIOUS_NEXT]` is the one option that does not fit:
previous/next are player controls on a live recorder. A recording scenario
wants no control group, or a template-appropriate one - the buttons as shipped
have nothing to act on.

Note also `startPip()` runs at page entry, not on backgrounding. That is
correct with auto-start enabled: `startPiP()` registers the window with the
system, which then shows it only when the app actually leaves the foreground.

**Stop and save on PiP close — same file** (corrected, see `HW-18-0088`)

```typescript
onStateChange(state: PiPWindow.PiPState, reason: string) {
  switch (state) {
    case PiPWindow.PiPState.ABOUT_TO_START:
      this.curState = 'ABOUT_TO_START';
      break;
    case PiPWindow.PiPState.STARTED:
      this.curState = 'STARTED';
      break;
    case PiPWindow.PiPState.ABOUT_TO_STOP:
      this.curState = 'ABOUT_TO_STOP';
      break;
    case PiPWindow.PiPState.STOPPED:
      this.curState = 'STOPPED';
      // FIX: the sample only logs here - the promised stop-and-save is absent
      if (this.recording) {
        this.recording = false;
        stopRecord().then((savedUri: string) => {
          videoUri = savedUri;
          hilog.info(DOMAIN, TAG, `Recording finalized at ${savedUri}`);
        });
      }
      break;
    case PiPWindow.PiPState.ABOUT_TO_RESTORE:
      this.curState = 'ABOUT_TO_RESTORE';
      break;
    case PiPWindow.PiPState.ERROR:
      this.curState = 'ERROR';
      break;
    default:
      break;
  }
  hilog.info(DOMAIN, TAG, `onStateChange: ${this.curState}, reason: ${reason}`);
}
```

**The document states the behaviour plainly - 当用户主动关闭画中画窗口时停止录像，
并将视频保存到相册 ("when the user actively closes the PiP window, stop
recording and save the video to the gallery") - and the shipped `STOPPED`
branch assigns a string and logs it.** Nothing on that path calls
`stopRecord()`, so `avRecorder.stop()` and `avRecorder.release()` never run,
the mp4 is never finalised, and the asset created by `createAsset` stays a
zero-length placeholder until the process dies (`HW-18-0088`). The bug is
particularly cruel because the recording *looks* fine right up to the moment
the user closes the window - which is the one gesture the document tells them
to make.

Guard the stop on the recording flag: `STOPPED` also fires on an ordinary
restore-then-close and must not call `stopRecord()` on a released recorder.

**The recording target file — `entry/src/main/ets/utils/VideoRecorder.ets`** (corrected, see `HW-18-0029`, `HW-18-0073`)

```typescript
export async function videoRecording(isStabilization: boolean, cameraPosition: number, qualityLevel: number,
  surfaceId: string, context: Context, callback?: (previewSize: Size) => void): Promise<number[]> {
  let zoomRatioRange: number[] = [];
  try {
    let options: photoAccessHelper.CreateOptions = { title: Date.now().toString() };
    let accessHelper: photoAccessHelper.PhotoAccessHelper = photoAccessHelper.getPhotoAccessHelper(context);
    let videoUri: string = await accessHelper.createAsset(photoAccessHelper.PhotoType.VIDEO, 'mp4', options);
    file = fileIo.openSync(videoUri, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
    let aVRecorderConfig: media.AVRecorderConfig = {
      audioSourceType: media.AudioSourceType.AUDIO_SOURCE_TYPE_MIC,
      videoSourceType: media.VideoSourceType.VIDEO_SOURCE_TYPE_SURFACE_YUV,
      profile: aVRecorderProfile,
      url: `fd://${file.fd.toString()}`,
      rotation: cameraPosition === 0 ? 90 : 270
    };
    uri = videoUri;
    avRecorder = await media.createAVRecorder();
    await avRecorder.prepare(aVRecorderConfig);
    let videoSurfaceId: string | undefined = await avRecorder.getInputSurface();

    videoOutput = cameraManager.createVideoOutput(videoProfile, videoSurfaceId);
    videoSession = cameraManager.createSession(camera.SceneMode.NORMAL_VIDEO) as camera.VideoSession;
    videoSession.beginConfig();
    videoSession.addInput(cameraInput);
    previewOutput = cameraManager.createPreviewOutput(previewProfile, surfaceId);  // FIX: no `let` - assign the module var
    videoSession.addOutput(previewOutput);
    videoSession.addOutput(videoOutput);
    await videoSession.commitConfig();
    await videoSession.start();
    zoomRatioRange = videoSession.getZoomRatioRange();
  } catch (error) {
    hilog.error(DOMAIN, TAG, `The videoRecording call failed. error: ${JSON.stringify(error)}`);
  }
  // FIX: the shipped code has `finally { fileIo.closeSync(file); }` here.
  //      The fd must stay open for the whole recording; stopRecord() closes it once.
  return zoomRatioRange;
}
```

**Two one-word defects, both fatal to the output.** The shipped function ends
with `finally { fileIo.closeSync(file); }`, which runs immediately after
`prepare()` - long before `avRecorder.start()` is ever called. The recorder
then writes to an fd the app has already closed, or to whatever unrelated file
inherited that number (`HW-18-0029`; `stopRecord()` closes it a *second* time,
and on an early-return path `file` may be unassigned, so the `finally` itself
throws). The official AVRecorder guide closes the file only after `stop()`.

The second is `let previewOutput` on the creation line, shadowing the module
variable of the same name. `stopRecordPreview()` guards
`if (previewOutput) { previewOutput.release(); }` against the module variable,
which was never assigned - so the preview output leaks on every record/stop
cycle and later session configs start failing (`HW-18-0073`, shared with
`AVRecorderTimer`). Dropping the `let` is the entire fix.

**Rotation at the moment recording starts — same file** (as shipped)

```typescript
export async function startRecord(): Promise<void> {
  // Update the rotation angle before starting recording
  const deviceDegree = await getGravity();
  try {
    await avRecorder.updateRotation(getVideoRotation(videoOutput, deviceDegree));
    await videoOutput.start();
    await avRecorder.start();
  } catch (error) {
    hilog.error(DOMAIN, TAG, `The startRecord call failed. error: ${JSON.stringify(error)}`);
  }
}
```

**This is the orientation pattern `PHOTO-29` should have used.** `getGravity()`
in `GravityUtil.ets` is a one-shot read - `sensor.once(sensor.SensorId.GRAVITY, ...)`
wrapped in a promise, with the same `(x*x + y*y) * 3 < z*z` flat-device guard -
so it resolves a degree without leaving a subscription behind. It is then
awaited *before* the recorder starts, so the value is real rather than a stale
field. `CameraTwist` instead starts a listener and reads a field on the next
line (`HW-18-0085`).

The degree is not used directly: `videoOutput.getVideoRotation(deviceDegree)`
converts a device degree into the rotation the *encoder* needs, which depends
on the sensor mounting of the specific camera. Never compute that mapping
yourself; the API exists because it differs per device.

`updateRotation` must be called before `start()` - the rotation is baked into
the container metadata at start time and cannot be changed mid-recording.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.CAMERA",          "reason": "$string:reason_camera",           "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.MICROPHONE",      "reason": "$string:reason_microphone",       "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.WRITE_IMAGEVIDEO","reason": "$string:reason_write_imagevideo", "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.READ_IMAGEVIDEO", "reason": "$string:reason_read_imagevideo",  "usedScene": { "abilities": ["EntryAbility"] } }
]
```

- Four declared; **five requested at runtime**. The extra
  `ohos.permission.MEDIA_LOCATION` is what breaks the grant (`HW-18-0007`).
- `MICROPHONE` is genuinely needed here: `AudioSourceType.AUDIO_SOURCE_TYPE_MIC`
  is part of the recorder config, unlike the photo-only samples in this
  industry.
- `READ_IMAGEVIDEO` / `WRITE_IMAGEVIDEO` are restricted (ACL). The write side
  goes through `createAsset`, which does not need the permission; the read side
  queries the album with `photoHelper.getAssets` for the thumbnail - use the
  picker instead and both can go (compare `HW-18-0004`).
- `usedScene` has no `when`. Background *recording* here is legitimised by the
  PiP window, not by a background permission; without PiP the platform would
  suspend the session. `deviceTypes` is `phone` only.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- PiP requires a `navigationId` when the page lives inside a `Navigation`; the
  sample passes `''` because it does not.
- The recorder profile is fixed at 48 kHz stereo AAC and a 32 Mbps video
  bitrate; the video profile search demands `frameRateRange.max === 60` for the
  back camera at quality level 0, so a device that only offers 30 fps at that
  size yields `videoProfile === undefined` and the config falls back to
  `undefined` width and height.
- The camera-switch button in the control panel is a UI stub - its `onClick`
  contains only a comment (此处为UI效果，无实际功能: "this is a UI effect with no
  actual function").
- The pinch upper clamp is a hardcoded `15`; the lower one compares against an
  unassigned `zoomRatioRange` (`HW-18-0023`).
- The thumbnail refresh is a 500 ms `setTimeout` after `stopRecord()`, not a
  callback - on a slow finalise it shows the previous video.

## Pitfalls

- **`HW-18-0007`** (D/high, confirmed, systematic): the runtime request array
  contains `ohos.permission.MEDIA_LOCATION`, which `module.json5` does not
  declare, so `requestPermissionsFromUser` rejects the entire call and neither
  CAMERA nor MICROPHONE is ever granted - the headline feature cannot start.
  Same defect in `AVRecorderTimer` (`PHOTO-25`). Fix: remove MEDIA_LOCATION
  from the array.
- **`HW-18-0088`** (D/medium, confirmed): the document promises that closing
  the PiP window stops recording and saves to the gallery; the `PiPState.STOPPED`
  branch of `onStateChange` only assigns a string and logs. The recording is
  lost. Fix: call `stopRecord()` on that path, guarded by the recording flag.
- **`HW-18-0029`** (B/high, confirmed, systematic): the AVRecorder target fd is
  closed in a `finally` right after `prepare()`, before recording starts, and a
  second time in `stopRecord()`; on early-exit paths `closeSync` runs on a
  possibly-unassigned `file` and throws from the `finally`. Fix: keep the file
  open until `avRecorder.stop()` completes and close it once.
- **`HW-18-0073`** (B/medium, confirmed): `let previewOutput` inside
  `videoRecording` shadows the module variable, making the release in
  `stopRecordPreview` dead code and leaking a preview output per cycle. Fix:
  drop the `let`.
- **`HW-18-0030`** (B/medium, confirmed, systematic): `requestPermissionsFromUser`'s
  result is ignored - `Index.ets:81-86` builds the pipeline on denial too, and
  the promise has no `.catch`. Fix: check `authResults` and add a refused-state
  UI.
- **`HW-18-0041`** (B/medium, probable, systematic): camera init is deferred by
  a fixed 200 ms timer with no check that `surfaceId` is non-empty. Fix: start
  from `onLoad` after the grant.
- **`HW-18-0010`** (B/medium, confirmed, systematic): `stopRecordPreview()`
  awaits nothing and nulls nothing, and `Index.ets:233-244` calls it and then
  `videoRecording(...)` without awaiting either - the rebuild races the
  teardown. Fix: await the teardown in order and reset the fields.
- **`HW-18-0023`** (B/low, confirmed, systematic): `zoomRatioRange` is never
  assigned - every call site discards `videoRecording`'s return value - so the
  pinch clamp compares against `undefined`. Fix: assign the returned range.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-pipwindow.md` - `PiPWindow.create`, `PiPController`, `PiPState`, `setAutoStartEnabled`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-pipwindow
- `documentation/harmonyos-references/04_media/arkts-apis-media-avrecorder.md` - `prepare`, `getInputSurface`, `updateRotation`, `start`, `stop`, and when to close the fd
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avrecorder
- `documentation/harmonyos-references/04_media/arkts-apis-camera-cameramanager.md` - `getSupportedOutputCapability`, `createVideoOutput`, `createSession`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-camera-cameramanager
- `documentation/harmonyos-references/04_media/arkts-apis-camera-videooutput.md` - `getVideoRotation`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-camera-videooutput
- `documentation/harmonyos-references/03_system/js-apis-sensor.md` - `sensor.once` and `GravityResponse`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-sensor
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `CAMERA`, `MICROPHONE`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - `READ/WRITE_IMAGEVIDEO`, `MEDIA_LOCATION`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `PHOTO-25` - `AVRecorderTimer`, the sibling recorder carrying the identical permission and fd defects
- `PHOTO-29` - the same orientation problem solved with a leaking listener instead of `sensor.once`
- `PHOTO-19` - `VideoEditPiPWindow`, PiP used for editing rather than capture
