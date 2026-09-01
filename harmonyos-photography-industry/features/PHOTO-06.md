---
id: PHOTO-06
title: Camera Kit + AVRecorder video capture - the recorder surface feeds the camera session, and the fd must outlive the recording
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/06_video_recording.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_recording-0000002258016882
sample: huawei_industry_tree/18_photography/downloads/VideoRecording.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CameraKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaKit", "@kit.PerformanceAnalysisKit"]
apis: [abilityAccessCtrl, bundleManager, camera, common, display, fileIo, hilog, image, media, preferences, window, "media.createAVRecorder", AVRecorderConfig, AVRecorderProfile, getInputSurface, "camera.getCameraManager", getSupportedOutputCapability, createVideoOutput, createPreviewOutput, createSession, commitConfig, XComponent, XComponentController, Video, VideoController, TextTimer, Navigation, AVImageGenerator, fetchFrameByTime, requestPermissionOnSetting]
permissions: [ohos.permission.MICROPHONE, ohos.permission.CAMERA]
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0029, HW-18-0030, HW-18-0031, HW-18-0032, HW-18-0005, HW-18-0090, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when the app has to **record video itself** rather than hand the
job to the system camera picker: an in-app camcorder, a video-note feature, a
clip recorder for a social feed, a supervised capture flow that must control
resolution, rotation and bitrate.

The pattern has one load-bearing idea. `AVRecorder` does not read from the
camera; it *publishes a surface*, and the camera writes into it. So the order
is fixed: create the recorder, `prepare()` it with a file destination, ask it
for `getInputSurface()`, then create the camera's `VideoOutput` **on that
surface id**. A second surface, from an `XComponent`, carries the preview.
Two surfaces, one session, one file.

**Read `HW-18-0029` before you copy anything from this sample.** The shipped
code closes the recording file descriptor in a `finally` block that runs
immediately after `prepare()` - before `start()` is ever called. Three
recorder samples in this industry share the defect. It is the difference
between a demo that looks right and a recorder that writes into a closed (or
recycled) fd. Everything else in this card is sound; that one block is not.

## Feature checklist

- A full-screen dark camera page: title bar, live preview on an `XComponent`,
  a running `TextTimer` overlay, and an action row.
- Idle state: switch-camera, record and open-list buttons. Recording state:
  pause/resume, stop and open-list; the timer runs, pauses and resets with the
  recorder.
- Tapping record checks CAMERA and MICROPHONE; if either is missing it walks
  the user through `requestPermissionOnSetting` and then a dialog that opens
  the system settings page for the app.
- Switching camera tears the session down, flips the stored position, rebuilds
  the recorder and re-prepares the camera.
- Stopping writes an MP4 named `VIDEO_<timestamp>.mp4` into the app sandbox
  (`filesDir`) and rebuilds the recorder ready for the next take.
- The list page shows a two-column grid of recordings, each with a first-frame
  thumbnail pulled by `AVImageGenerator.fetchFrameByTime`.
- The play page plays the sandbox file in a `Video` component with that
  thumbnail as `previewUri`, plus a play/pause button, elapsed/total time and a
  seek slider.

## Architecture

One `entry` module. All recorder and camera state lives in a single exported
singleton; the pages hold only UI state, and cross-page state travels through
`AppStorage`.

```
entry/src/main/ets
├── constants/
│   ├── CameraConstants.ets       CAMERA_POSITION_FRONT = 1, _BACK = 0
│   └── StyleConstants.ets        sizes, plus PathConstants for the Navigation route names
├── entryability/EntryAbility.ets dark color mode, context handoff, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── model/VideoListItem.ets       { pixelMap, name }
├── pages/
│   ├── MainPage.ets              @Entry: Navigation root, preview, timer, action row
│   ├── VideoList.ets             NavDestination: thumbnail grid over filesDir
│   └── VideoPlay.ets             NavDestination: Video + slider
└── util/
    ├── GlobalContext.ets         context holder for the preferences store
    ├── Logger.ets
    ├── PermissionsRequest.ets    check / request / request-on-setting / open settings
    └── VideoUtils.ets            the singleton: recorder + camera pipeline, 517 lines
```

The documented 工程目录 matches the zip exactly, including
`EntryBackupAbility.ets`.

**The design decision worth copying** is the split between `VideoUtils` and
`AppStorage`. `VideoUtils` is `export default new VideoUtils()` - one instance,
holding the six native objects (`avRecorder`, `cameraManager`,
`cameraSession`, `videoOutput`, `cameraInput`, `previewOutput`) that must never
be duplicated. Everything the *UI* needs to know about them is published as
scalars in `AppStorage`: `recording`, `pausing`, `isFinished`, `cameraPosition`,
`surfaceId`. `MainPage` binds those with `@StorageLink` and never holds a
camera object. That is why `stopRecordingProcess()` can flip the button row
from inside the utility without a callback, and why the recorder survives a
navigation to the list page and back.

**The decision worth avoiding** is that `prepareCamera()` is a 180-line linear
function in which every step has its own `try`/`catch` that logs and then
*falls through*. A failed `commitConfig` is logged and `start()` is called
anyway. If you lift this file, make the catches return.

## Implementation steps

1. **Declare CAMERA and MICROPHONE** with `reason` and `usedScene`. This
   sample uses `"when": "always"`; `"inuse"` is the honest value for a
   foreground recorder with no continuous task.
2. **Check the permission result before building anything** (`HW-18-0030`):
   `requestPermissionsFromUser` resolves even on a refusal, so read
   `data.authResults` and add a `.catch`.
3. **Create the recorder before the camera.** `media.createAVRecorder()`,
   register the `stateChange` and `error` callbacks, then `prepare()`.
4. **Open the destination file and keep it open.** Build the sandbox path,
   `fs.openSync(..., READ_WRITE | CREATE)`, pass `url: fd://${file.fd}`. **Do
   not close it in a `finally` after `prepare()`** (`HW-18-0029`) - close it
   once, after `stop()` has resolved.
5. **Match the profile to the camera.** `AVRecorderProfile.videoFrameWidth` /
   `videoFrameHeight` must equal the `camera.VideoProfile` you select, or the
   session will not commit. Set `rotation` from the camera position - 90 back,
   270 front; `prepare()` rejects anything but 0/90/180/270.
6. **Take the recorder's surface** with `getInputSurface()` and pass it to
   `cameraManager.createVideoOutput(videoProfile, videoSurfaceId)`.
7. **Take the preview surface from the `XComponent`** in its `onLoad`, publish
   it to `AppStorage`, and pass it to `createPreviewOutput`. Keep the
   `XComponent`'s surface aspect ratio equal to the profile's (16:9 here) or
   the preview will be stretched.
8. **Assemble the session**: `createSession(NORMAL_VIDEO)`, `beginConfig`,
   `addInput`, `addOutput(preview)`, `addOutput(video)`, `commitConfig`,
   `start`, then `videoOutput.start()`.
9. **Compare optional numbers against `undefined`, not truthiness**
   (`HW-18-0031`): the back camera is position `0`, which is falsy.
10. **Guard the seek/update conflict on playback** (`HW-18-0032`): toggle a
    flag on slider touch-down and touch-up so `onUpdate` does not fight the
    drag.

## Verified snippets

All snippets are from `VideoRecording.zip`. Corrected forms are marked.

**Recorder preparation - `entry/src/main/ets/util/VideoUtils.ets`** (corrected, see `HW-18-0029`)

```typescript
private async prepareAVRecorder() {
  if (this.avRecorder !== undefined && this.avRecorder.state === 'idle') {
    try {
      const fileName: string =
        VideoUtils.context.filesDir + '/' + 'VIDEO_' + Date.parse(new Date().toString()) + '.mp4';
      const curFile = fs.openSync(fileName, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);

      const cameraPosition = AppStorage.get('cameraPosition') as number;
      let rotate: number = 90;                    // 后置摄像头旋转90度
      if (cameraPosition === 1) {
        rotate = 270;                             // 前置摄像头旋转270度
      }

      const avProfile: media.AVRecorderProfile = {
        audioBitrate: 100000, audioChannels: 2, audioSampleRate: 48000,
        audioCodec: media.CodecMimeType.AUDIO_AAC,
        fileFormat: media.ContainerFormatType.CFT_MPEG_4,
        videoBitrate: 2000000, videoFrameRate: 30,
        videoCodec: media.CodecMimeType.VIDEO_AVC,
        videoFrameWidth: this.imageSize.width,      // must equal the camera VideoProfile
        videoFrameHeight: this.imageSize.height
      };

      const aVRecorderConfig: media.AVRecorderConfig = {
        audioSourceType: media.AudioSourceType.AUDIO_SOURCE_TYPE_MIC,
        videoSourceType: media.VideoSourceType.VIDEO_SOURCE_TYPE_SURFACE_YUV,
        profile: avProfile,
        url: `fd://${curFile.fd}`,                  // the recorder writes here until stop()
        rotation: rotate
      };

      await this.avRecorder.prepare(aVRecorderConfig);
      this.videoSurfaceId = await this.avRecorder.getInputSurface();
      AppStorage.setOrCreate('videoSurfaceId', this.videoSurfaceId);
      this.curFile = curFile;
      // FIX: the sample has `finally { if (curFile) { fs.closeSync(curFile.fd); } }` here.
      //      The fd must stay open for the whole recording; it is closed in stopRecordingProcess.
    } catch (err) {
      let error = err as BusinessError;
      logger.error(`AVRecorderVideoDemo prepareAVRecorder error message:${error.message}`);
    }
  }
}
```

and the matching close, in `stopRecordingProcess`:

```typescript
if (this.avRecorder.state === 'started' || this.avRecorder.state === 'paused') {
  try {
    await this.avRecorder.stop();
    await this.stopCameraOutput();
  } catch (error) {
    let err = error as BusinessError;
    logger.error(`AVRecorderVideoDemo avRecorder stop error: ${JSON.stringify(err)}`);
  }
}
if (this.curFile !== undefined) {            // FIX: close exactly once, after stop() resolved
  fs.closeSync(this.curFile.fd);
  this.curFile = undefined;
}
await this.avRecorder.reset();
await this.releaseAVRecorder();
await this.createAVRecorder();               // ready for the next take
```

**`url: fd://<n>` is a live handle, not a path.** `prepare()` records the
integer; the muxer writes MP4 boxes into that descriptor for the entire
recording and only finalises the `moov` atom on `stop()`. Closing it in a
`finally` right after `prepare()` - which is what the shipped code does -
leaves the recorder holding a number the process no longer owns. The best case
is that the native layer duplicated the descriptor and nothing is observable;
the realistic case is a truncated or empty MP4; the worst case is that the
number gets reused by the next `fs.openSync` (the thumbnail generator in
`VideoList` opens files with the same API) and the recorder writes video frames
into an unrelated file. The same block appears verbatim in `AVRecorderTimer`
and `PipwindowRecorder`, both of which additionally close the same fd a
*second* time in their stop path, and both of which run `closeSync` on a
possibly-unassigned `file` on early-exit paths - so the `finally` itself
throws and hides the original error.

Two smaller points. `getInputSurface()` is called twice in the shipped code -
once into `this.videoSurfaceId` and once inside the `AppStorage.setOrCreate`
argument; call it once, because there is no contract that a second call returns
the same id. And `rotation` here is derived from the camera position rather
than hardcoded, which is more than `PHOTO-04` and `PHOTO-07` manage for stills
(`HW-18-0027`).

**Wiring the two surfaces into one session - same file** (as shipped, trimmed)

```typescript
async prepareCamera(cameraPosition: number) {
  // ... getSupportedCameras / getSupportedOutputCapability(devices[pos], SceneMode.NORMAL_VIDEO)

  // 音视频录制参数设置 - the recorder is prepared first, so its surface exists
  await this.prepareAVRecorder();

  // videoProfile的宽高需要与AVRecorderProfile的宽高保持一致
  let videoProfile: undefined | camera.VideoProfile = videoProfilesArray.find((profile: camera.VideoProfile) => {
    return profile.size.width === this.imageSize.width && profile.size.height === this.imageSize.height;
  });
  if (!videoProfile) {
    logger.error(`AVRecorderVideoDemo videoProfile is not found`);
    return;
  }

  this.videoOutput = this.cameraManager.createVideoOutput(videoProfile, this.videoSurfaceId);
  this.videoOutput.on('error', (error: BusinessError) => { /* ... */ });

  this.cameraSession = this.cameraManager.createSession(camera.SceneMode.NORMAL_VIDEO) as camera.VideoSession;
  this.cameraSession.beginConfig();

  this.cameraInput = this.cameraManager.createCameraInput(cameraDevices[cameraPosition]);
  await this.cameraInput.open();
  this.cameraSession.addInput(this.cameraInput);

  // previewProfile is found the same way, then bound to the XComponent's surface
  this.previewOutput = this.cameraManager.createPreviewOutput(previewProfile, AppStorage.get('surfaceId'));

  this.cameraSession.addOutput(this.previewOutput);      // XComponent surface
  this.cameraSession.addOutput(this.videoOutput);        // AVRecorder surface
  await this.cameraSession.commitConfig();
  await this.cameraSession.start();

  this.videoOutput.start((err: BusinessError) => { /* ... */ });
}
```

**The ordering is the API contract, not a style choice.** `prepareAVRecorder()`
must run before `createVideoOutput`, because `getInputSurface()` is only valid
in the `prepared` state and the surface id is a constructor argument to the
video output. `beginConfig` / `addInput` / `addOutput` / `commitConfig` is a
transaction - outputs added after `commitConfig` are ignored, and a session
committed with mismatched profile sizes fails. And `videoOutput.start()` is
separate from `cameraSession.start()`: the session starts the *pipeline*, the
video output starts *pushing frames into the recorder surface*. The recorder
itself is still idle at this point; `avRecorder.start()` happens later, when
the user presses record.

Note that the preview surface comes from `AppStorage.get('surfaceId')`, written
by the `XComponent`'s `onLoad` in `MainPage`, and that the same `onLoad` sets
`surfaceHeight = width * 16 / 9` to match the 1920x1080 profile.

**Permission gate and recorder recovery - `pages/MainPage.ets` and `util/VideoUtils.ets`** (corrected, see `HW-18-0030`, `HW-18-0031`)

```typescript
// MainPage.onPageShow
async onPageShow(): Promise<void> {
  abilityAccessCtrl.createAtManager()
    .requestPermissionsFromUser(this.context, this.cameraPermission)
    .then(async (data: PermissionRequestResult) => {
      // FIX: the sample ignores `data` entirely and builds the pipeline regardless
      const granted = data.authResults.every((r: number) => r === 0);
      if (!granted) {
        return;                                   // leave the preview empty; the record button re-asks
      }
      await VideoUtils.createAVRecorder();
      VideoUtils.createCameraManager();
      await VideoUtils.prepareCamera(this.cameraPosition);
    })
    .catch((err: BusinessError) => {              // FIX: absent in the sample
      logger.error(`requestPermissionsFromUser failed: ${err.message}`);
    });
}

// VideoUtils.startRecordingProcess
async startRecordingProcess() {
  try {
    if (this.avRecorder === undefined) {
      await this.createAVRecorder();
      let cameraPosition: number | undefined = AppStorage.get('cameraPosition');
      if (cameraPosition !== undefined) {         // FIX: the sample writes `if (cameraPosition)`
        await this.prepareCamera(cameraPosition);
      }
    }
    if (this.avRecorder !== undefined) {
      if (this.avRecorder.state === 'prepared') {
        await this.avRecorder.start();
      }
      AppStorage.setOrCreate('recording', true);
      AppStorage.setOrCreate('pausing', false);
      AppStorage.setOrCreate('isFinished', false);
    }
  } catch (error) {
    let err = error as BusinessError;
    logger.error('AVRecorderVideoDemo startRecordingProcess' + JSON.stringify(err));
  }
}
```

**`if (cameraPosition)` is the falsy-zero trap in its purest form.** The back
camera is position `0` and it is the default, so on the recovery path - where
`avRecorder` came back `undefined` and the pipeline has to be rebuilt - the
most common camera is precisely the one that gets skipped, and `start()` is
then called on a recorder with no camera feeding its surface. `!== undefined`
is the only correct test for an `AppStorage.get<number>` result.

The permission gap is worth understanding because the sample *does* have a
correct flow, just not here: the record button calls
`PermissionsRequest.checkPermissions()`, and on refusal walks through
`requestPermissionsOnSetting` and a dialog that opens the app's settings page.
`onPageShow` bypasses all of it and builds the camera pipeline in a bare
`.then()`, so a user who denies the dialog gets a page full of raw camera
errors before ever reaching the good path. The same shape appears in six
photography samples (`HW-18-0030`).

**Playback seek guard - `entry/src/main/ets/pages/VideoPlay.ets`** (corrected, see `HW-18-0032`)

```typescript
@State private seeking: boolean = false;   // FIX: shipped as `private seeking: boolean = true`

Video({ src: this.filePath, controller: this.controller, previewUri: this.firstFramePicture })
  .controls(false)
  .onPrepared((event) => {
    this.duration = event.duration;
  })
  .onUpdate((event) => {
    if (!this.seeking) {                   // FIX: shipped as `if (this.seeking)`, always true
      this.currentTime = event.time;
    }
  })
  .onFinish(() => {
    this.videoPausing = true;
  });

Slider({ value: this.currentTime, min: 0, max: this.duration, step: 1, style: SliderStyle.OutSet })
  .onChange((value: number, mode: SliderChangeMode) => {
    this.currentTime = value;
    if (mode === SliderChangeMode.Begin) {
      this.seeking = true;                 // FIX: nothing ever set the flag in the sample
    } else if (mode === SliderChangeMode.End) {
      this.controller.setCurrentTime(value);
      this.seeking = false;
    }
  });
```

**The comment in the shipped file documents an intent the code inverts.**
`private seeking: boolean = true; // 防止拖动时进度冲突` (prevent progress
conflicts while dragging) initialises the guard to the value that disables it,
and nothing ever writes it again - so `onUpdate` overwrites `currentTime` on
every frame, including while the user's finger is on the slider. Reading the
`SliderChangeMode` gives both edges for free, and seeking on `End` rather than
on every `Moving` event avoids hammering `setCurrentTime` during the drag.

`previewUri: this.firstFramePicture` is the `PixelMap` produced in `VideoList`
by `AVImageGenerator.fetchFrameByTime` and passed through the `NavPathStack`
parameter, so the page shows the right first frame before the decoder has
produced anything.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.MICROPHONE",
    "reason": "$string:permission_reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
  },
  {
    "name": "ohos.permission.CAMERA",
    "reason": "$string:permission_reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
  }
]
```

- Both are `user_grant`, so `reason` and `usedScene` are mandatory and the
  reason string resource must exist.
- `"when": "always"` claims background use. This app records only in the
  foreground and has no continuous task, so `"inuse"` is the correct value;
  `"always"` invites extra scrutiny in review for no functional gain.
- `AUDIO_SOURCE_TYPE_MIC` in `AVRecorderConfig` is what actually makes
  MICROPHONE mandatory - a mute recorder would drop both the source type and
  the permission.
- `routerMap: "$profile:route_map"` backs the `Navigation` routes
  (`VideoList`, `VideoPlay`) rather than a page router.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **1920x1080 at 30fps is hardcoded** in `imageSize` and used to `find` both
  the video and the preview profile. On a device that does not expose exactly
  that pair, `prepareCamera` logs `videoProfile is not found` and returns, and
  the page shows a black preview with no error. Fall back to the first
  supported profile, or use `camera` preconfig.
- Recordings go to `filesDir`, the app sandbox - they are **not** in the
  gallery and disappear with the app. Adding them to the album needs
  `MediaAssetChangeRequest` or the `SaveButton` flow (`PHOTO-05`, `PHOTO-08`).
- Every step of `prepareCamera` catches, logs and continues. A failed
  `commitConfig` still reaches `session.start()`.
- `releaseCamera()` awaits each teardown step in the correct order (stop,
  close, release preview, release video, release session), which makes it the
  *good* example against `HW-18-0010` - but it never nulls the fields, so the
  `!== undefined` guards stay true on released objects.
- `VideoList.FetchFrameByTimeFromFile` swallows its `catch` entirely and runs
  `fs.closeSync(fd)` in the `finally` on an `fd` declared `null!`, so an
  `openSync` failure throws a `TypeError` out of the `finally`. Its
  `PixelMapParams` also asks for 2320x1080 for a 1920x1080 recording.

## Pitfalls

- **`HW-18-0029`** (B/high, confirmed): **systematic - the recorder's target fd
  is closed immediately after `prepare()`, before recording starts.**
  `VideoUtils.prepareAVRecorder` opens `curFile`, sets `url: fd://${curFile.fd}`,
  awaits `prepare`, then closes the fd in a `finally`; `start()` happens later.
  `AVRecorderTimer` (`VideoRecorder.ets:116-121`, `:184-188`) and
  `PipwindowRecorder` (`VideoRecorder.ets:188-192`) are identical, and both
  close the same fd a *second* time in `stopRecord`, and both call `closeSync`
  on a possibly-unassigned `file` on early-exit paths so the `finally` itself
  throws. Result: the recorder writes to a descriptor the app has released -
  empty or corrupt MP4s on the main record path, or writes into an unrelated
  file if the number has been reused. Fix: keep the file open until
  `avRecorder.stop()` completes, close it exactly once there, and guard the
  `finally` for unassigned files.
- **`HW-18-0030`** (B/medium, confirmed): systematic - the
  `requestPermissionsFromUser` result is ignored in six samples
  (`VideoRecording MainPage.ets:48`, `CustomisedCamera`, `Picture`,
  `AVRecorderTimer`, `PipwindowRecorder`, `CameraResolution`), so the camera
  and recorder pipeline is built even when the user denies, and the rejection
  is unhandled. Fix: check `data.authResults` before building, add `.catch`.
- **`HW-18-0031`** (B/low, confirmed): `if (cameraPosition)` in
  `startRecordingProcess` skips `prepareCamera` for the back camera, which is
  position `0` and the default. On the recorder-recovery path recording then
  starts with no camera session. Fix: `if (cameraPosition !== undefined)`.
- **`HW-18-0032`** (B/low, confirmed): `VideoPlay`'s `seeking` flag is
  initialised `true` and never toggled, so the drag-conflict guard its own
  comment describes is dead and `onUpdate` overwrites `currentTime` mid-drag.
  Fix: toggle it on `SliderChangeMode.Begin` / `End`.
- **`HW-18-0005`** (B/low, confirmed): systematic - `ImageSource` instances
  created and never released across eight photography files. The finding lists
  `VideoRecording MainPage.ets` among them; in the zip we extracted,
  `MainPage.ets` contains no `createImageSource` call and the only native media
  object it creates indirectly (`AVImageGenerator`, in `VideoList.ets`) *is*
  released. Treat the rule as binding for any `ImageSource` you add here, but
  the instance attributed to this sample could not be confirmed.
- **`HW-18-0010`** (B/medium, confirmed) does **not** hit this sample:
  `releaseCamera()` awaits every teardown step. Use it as the reference
  implementation when fixing `CustomisedCamera` (`PHOTO-07`) and the seven
  other samples that do not.

## References

- `huawei_industry_tree/18_photography/docs/06_video_recording.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/video_recording-0000002258016882
- `documentation/harmonyos-references/04_media/arkts-apis-media-avrecorder.md` - `AVRecorder`, `AVRecorderConfig`, the state machine
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avrecorder
- `documentation/harmonyos-guides/05_media/using-avrecorder-for-recording.md` - the prepare/start/stop lifecycle and file handling
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/using-avrecorder-for-recording
- `documentation/harmonyos-guides/05_media/camera-recording-case.md` - the full camera-plus-recorder wiring
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-recording-case
- `documentation/harmonyos-references/04_media/js-apis-camera.md` - `VideoSession`, `VideoOutput`, `VideoProfile`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-camera
- `documentation/harmonyos-references/04_media/arkts-apis-camera-cameramanager.md` - `getSupportedOutputCapability`, `createVideoOutput`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-camera-cameramanager
- `documentation/harmonyos-guides/05_media/camera-session-management.md` - `beginConfig` / `commitConfig` as a transaction
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-session-management
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - the preview surface
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-references/02_application-framework/ts-media-components-video.md` - `Video`, `previewUri`, `onUpdate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-media-components-video
- `documentation/harmonyos-guides/05_media/avimagegenerator.md` - `fetchFrameByTime` for the thumbnails
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/avimagegenerator
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - CAMERA, MICROPHONE
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `PHOTO-07` - the stills sibling: same camera setup, `PhotoSession` instead of `VideoSession`
- `PHOTO-05` - getting media out of the sandbox and into the gallery
