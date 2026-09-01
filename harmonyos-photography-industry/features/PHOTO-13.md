---
id: PHOTO-13
title: Capture timer - a countdown shutter over a custom camera preview, with a second permission ask
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/13_capture_timer.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/capture_timer-0000002352218316
sample: huawei_industry_tree/18_photography/downloads/CaptureTimer.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CameraKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [abilityAccessCtrl, bundleManager, camera, common, curves, display, hilog, image, photoAccessHelper, window]
permissions: [ohos.permission.CAMERA]
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0041, HW-18-0042, HW-18-0010, HW-18-0021, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when you are building a **custom camera screen with a self-timer**
- the shutter that waits 3 or 7 seconds, shows a shrinking number over the live
preview, then fires. It is the standard "group photo" affordance in camera and
beautification apps, and the same three pieces (a mode toggle, a visible
countdown, a delayed capture) recur in video recording, screen recording and
burst-shot controls.

The pattern is deliberately small: the timer is **not** a camera feature. It is
one number in page state, a `TextTimer` bound to that number, and a `setTimeout`
that calls `photoOutput.capture()` when it expires. That separation is the
transferable part - you can add a self-timer to an existing camera page without
touching the session code.

The sample also demonstrates the **permission second chance**: CAMERA is
`user_grant` with no fallback UI, so on refusal the app shows a button that
calls `requestPermissionOnSetting` and rebuilds the preview when the grant comes
back. **Read `HW-18-0041` before copying the initialisation flow** - the sample
starts the camera from three places and two of them run before the surface
exists.

## Feature checklist

- The camera preview fills a fixed-height `XComponent` on a black page.
- A clock icon in the top left cycles 0 -> 3s -> 7s, its glyph changing with the
  selection, and flashes a label ("倒计时已关闭" / "倒计时 Ns") for one second.
- Tapping the shutter with a non-zero timer hides the bottom controls and counts
  down on screen with a large white number.
- At zero the photo is captured, a black flash animates over the preview, and
  the controls come back.
- The captured photo appears as a round thumbnail in the bottom left; tapping it
  opens the system Photos app on that asset.
- A switch button flips between the back and front camera; front captures are
  mirrored.
- If CAMERA is refused, the preview is replaced by a single button that opens the
  permission settings sheet; granting there brings the preview back.
- Leaving the page releases the camera; returning to it re-checks the permission
  and re-initialises.

## Architecture

One `entry` module. The page owns all state via `@Provide`, five components
`@Consume` it, and one utility class owns the camera pipeline.

```
entry/src/main/ets
├── component
│   ├── FlashBlackComponent.ets                the black shutter flash, driven by a counter
│   ├── PictureComponent.ets                   thumbnail + shutter + camera-switch row
│   ├── StackXComponent.ets                    XComponent preview, stacked with countdown + flash
│   ├── TimerShootingComponent.ets             two structs: the clock toggle and the countdown text
│   └── TwiceReqPermissionButtonComponent.ets  the refusal screen and requestPermissionOnSetting
├── constants/CameraConstants.ets              sizes, profile constants, camera positions
├── entryability/EntryAbility.ets              full screen + avoid areas -> AppStorage
├── entrybackupability/
├── pages/Index.ets                            @Entry, all @Provide state, permission entry
└── utils
    ├── CameraUtils.ets                        the whole camera pipeline (183 lines)
    └── Logger.ets
```

The documented tree matches the zip exactly, including the file comments.

**The design decision worth copying** is that the shutter delay lives in
`PictureComponent`, not in `CameraUtils`. `CameraUtils.capture(isFront)` is
synchronous and unconditional; the component decides *when* to call it:

```typescript
setTimeout(() => {
  this.cameraUtils.capture(this.isFront);
  this.captureClickFlag = this.captureClickFlag + 1;
  this.isVisibleCapture = true;
}, this.captureTimer * 1000);
```

With `captureTimer === 0` that is a zero-delay timeout, so the no-timer path and
the timed path are the same code. Nothing in the camera layer branches on the
timer at all.

**The decision worth avoiding** is the shared `CameraUtils` passed through
`AppStorage`. `Index` publishes one with `AppStorage.setOrCreate('cameraUtils', ...)`,
and three components each construct a throwaway instance in a field initialiser
before overwriting it in `aboutToAppear` with an unchecked `as CameraUtils` cast.
`@Provide`/`@Consume` of the utility would be typed and would not construct
anything twice.

## Implementation steps

1. **Declare `ohos.permission.CAMERA`** in `module.json5` with `reason` and
   `usedScene`. It is `user_grant`, so the string resource must exist.
2. **Request the permission once in `aboutToAppear`,** and re-check it in
   `onPageShow` so a grant made in Settings while the app was backgrounded is
   picked up. Note the shipped `onPageShow` guard is inverted in effect: it only
   re-checks when `notHasPermission` is already `false`, so the refusal screen
   never re-checks by itself.
3. **Give the refusal state its own screen**, not a toast. `requestPermissionsFromUser`
   stops showing a dialog after a permanent refusal, so the button must call
   `requestPermissionOnSetting`, which opens the settings sheet directly.
4. **Initialise the camera from exactly one place - `XComponent.onLoad` - and
   only when the surface id is non-empty** (`HW-18-0041`). The permission
   callbacks should flip `notHasPermission` and nothing else; the XComponent
   they reveal will call `onLoad` and start the pipeline with a real surface.
5. **Do all capability checks before `cameraInput.open()`,** or close the input
   on every early return (`HW-18-0042`).
6. **Guard the `find()` results for the preview and photo profiles** before
   passing them to `createPreviewOutput` / `createPhotoOutput`, and fall back to
   the first supported profile (`HW-18-0021`).
7. **Await the teardown before rebuilding.** `releaseCamera` must await
   `stop -> close -> release` in order and null the fields, and the camera-switch
   handler must await it before the next `cameraShooting` (`HW-18-0010`).
8. **Cycle the timer with a modulo over parallel arrays** - icons and seconds
   share one index - and show the selection label for 1000 ms.
9. **Bind `TextTimer` with `isCountDown: true` and `count: seconds * 1000`,**
   start it from `onAppear`, reset it in `onTimer` at the target.
10. **Delay the capture with `setTimeout`,** and bump a counter the flash
    component `@Watch`es, so the animation is triggered by state rather than by
    a call across components.

## Verified snippets

All snippets are from `CaptureTimer.zip`. Corrected forms are marked.

**The timer toggle — `entry/src/main/ets/component/TimerShootingComponent.ets`** (as shipped)

```typescript
@Component
struct TimeSettingRowComponent {
  @Consume timerShooting: number;
  @Consume isVisibleTimerSet: boolean;
  @Consume isVisibleCapture: boolean;
  @State timerPic: Resource = $r('app.media.clock_close');
  timerImageArray: Resource[] =
    [$r('app.media.clock_close'), $r('app.media.clock_3s'), $r('app.media.clock_7s')];
  timerArray: number[] = [0, 3, 7];
  private index: number = 0;

  build() {
    Row() {
      Image(this.timerPic)
        .height(CameraConstants.IMAGE_HEIGHT)
        .margin(CameraConstants.MARGIN)
        .onClick(() => {
          this.index = (this.index + 1) % this.timerArray.length;
          this.timerPic = this.timerImageArray[this.index];
          this.timerShooting = this.timerArray[this.index];
          this.isVisibleTimerSet = true;
          setTimeout(() => {
            this.isVisibleTimerSet = false;
          }, 1000);
        })
        .visibility(this.isVisibleCapture ? Visibility.Visible : Visibility.Hidden);
    }
  }
}
```

**Two parallel arrays and one private index** is the whole mechanism. `index` is
deliberately *not* `@State` - it is an implementation detail of the cycle, and
the two values the UI reacts to (`timerPic`, `timerShooting`) are assigned
explicitly. `timerShooting` is `@Consume`, so it reaches the shutter in
`PictureComponent` without either component knowing about the other.
`isVisibleTimerSet` is a one-second-lived flag rather than a value, and the
`.visibility(...)` binding hides the toggle without unmounting it, so the layout
does not shift mid-countdown.

**The on-screen countdown — same file** (as shipped)

```typescript
@Component
struct TimerTextComponent {
  @Consume isVisibleTimerSet: boolean;
  @Consume timerShooting: number;
  @Consume captureTimer: number;
  @Consume isVisibleTimer: boolean;
  format: string = 's';
  textTimerController: TextTimerController = new TextTimerController();

  build() {
    Column() {
      if (this.isVisibleTimerSet && this.timerShooting === 0) {
        Text($r('app.string.countdown_closed_text'));      // 倒计时已关闭
      } else if (this.isVisibleTimerSet && this.timerShooting !== 0) {
        Text($r('app.string.countdown_text', this.timerShooting));
      }

      if (this.captureTimer !== 0 && this.isVisibleTimer) {
        TextTimer({ isCountDown: true, count: this.captureTimer * 1000, controller: this.textTimerController })
          .format(this.format)
          .fontSize(50)
          .onAppear(() => {
            this.textTimerController.start();
          })
          .onTimer((utc: number, elapsedTime: number) => {
            if (elapsedTime >= this.captureTimer) {
              this.textTimerController.reset();
              this.isVisibleTimer = false;
            }
          });
      }
    };
  }
}
```

**`TextTimer` counts, `setTimeout` fires.** They are two independent clocks
started at the same moment, which is why the countdown is purely decorative and
a dropped frame in the text cannot delay the shutter. `count` is in
milliseconds and `format: 's'` renders whole seconds; `isCountDown: true` makes
it run down from `count` rather than up from zero.

Starting the controller in `onAppear` rather than in the click handler keeps
this correct: the component only exists while `isVisibleTimer` is true, so its
appearance *is* the start event. Without the `onTimer` reset the controller
keeps its finished value and the next countdown starts from the wrong number.

**Camera initialisation, gated on the surface — `entry/src/main/ets/component/StackXComponent.ets`
and `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-18-0041`)

```typescript
// StackXComponent.ets - the single owner of camera start-up
XComponent({
  type: XComponentType.SURFACE,
  controller: this.mXComponentController,
  imageAIOptions: {
    types: [ImageAnalyzerType.SUBJECT],
    aiController: this.aiController
  }
})
  .onLoad(async () => {
    this.mXComponentController.setXComponentSurfaceRect({
      surfaceWidth: this.imageSize.width,
      surfaceHeight: this.imageSize.height
    });
    this.surfaceId = this.mXComponentController.getXComponentSurfaceId();
    if (!this.notHasPermission && this.surfaceId !== '') {     // FIX: guard the surface id
      this.cameraUtils.cameraShooting(this.cameraPosition, this.surfaceId, this.getUIContext().getHostContext()!);
    }
  })
  .height(CameraConstants.XCOMPONENT_HEIGHT)
  .hitTestBehavior(HitTestMode.Block);

// Index.ets - the permission callback only flips the flag
aboutToAppear(): void {
  AppStorage.setOrCreate('cameraUtils', this.cameraUtils);
  let atManager: abilityAccessCtrl.AtManager = abilityAccessCtrl.createAtManager();
  let context: Context = this.getUIContext().getHostContext() as common.UIAbilityContext;
  atManager.requestPermissionsFromUser(context, ['ohos.permission.CAMERA'])
    .then((data: PermissionRequestResult) => {
      this.notHasPermission = data.authResults[0] !== 0;      // FIX: no cameraShooting here -
    })                                                        // onLoad owns initialisation
    .catch((err: BusinessError) => {
      Logger.error(`data: ${JSON.stringify(err)}`);
    });
}
```

**`surfaceId` starts as `''` and only `onLoad` ever fills it.** In the shipped
code the grant callback in `aboutToAppear` calls `cameraShooting` immediately -
but the `XComponent` that produces the surface is inside the `else` branch of
`if (this.notHasPermission)`, so it has not been built yet. That first call
reaches `createPreviewOutput(previewProfile, '')` and fails, and moments later
`onLoad` fires a second, correct initialisation whose first act is an
un-awaited `releaseCamera()` of the half-open first attempt. The visible symptom
is an intermittent black preview on the first run after granting.

The same mistake sits in `TwiceReqPermissionButtonComponent`, which calls
`cameraShooting` from the `requestPermissionOnSetting` callback with the same
empty surface id. Both callbacks should do one thing - set
`notHasPermission = false` - and let the XComponent that appears start the camera.

**The pipeline, with the checks before `open()` — `entry/src/main/ets/utils/CameraUtils.ets`**
(corrected, see `HW-18-0042`, `HW-18-0021`, `HW-18-0010`)

```typescript
async cameraShooting(cameraPosition: number, surfaceId: string, context: Context) {
  await this.releaseCamera();                                 // FIX: shipped call is not awaited

  let cameraManager: camera.CameraManager = camera.getCameraManager(context);
  if (!cameraManager) {
    return;
  }
  let cameraArray: camera.CameraDevice[] = cameraManager.getSupportedCameras();
  if (cameraArray.length <= 0) {
    return;
  }

  // FIX: every capability check moved ahead of cameraInput.open()
  let sceneModes: camera.SceneMode[] = cameraManager.getSupportedSceneModes(cameraArray[cameraPosition]);
  if (sceneModes.indexOf(camera.SceneMode.NORMAL_PHOTO) < 0) {
    return;
  }
  let cameraOutputCap: camera.CameraOutputCapability =
    cameraManager.getSupportedOutputCapability(cameraArray[cameraPosition], camera.SceneMode.NORMAL_PHOTO);
  if (!cameraOutputCap) {
    return;
  }

  let previewProfile: undefined | camera.Profile = cameraOutputCap.previewProfiles.find(
    (profile: camera.Profile) => {
      return profile.size.width === this.previewProfileObj.size.width &&
        profile.size.height === this.previewProfileObj.size.height &&
        profile.format === this.previewProfileObj.format;
    }) ?? cameraOutputCap.previewProfiles[0];                 // FIX: fall back, never pass undefined
  let photoProfile: undefined | camera.Profile =
    cameraOutputCap.photoProfiles.find(/* same three comparisons on photoProfileObj */) ??
    cameraOutputCap.photoProfiles[0];
  if (!previewProfile || !photoProfile) {
    return;
  }

  this.cameraInput = cameraManager.createCameraInput(cameraArray[cameraPosition]);
  await this.cameraInput.open();                              // now the last point of no return

  this.previewOutput = cameraManager.createPreviewOutput(previewProfile, surfaceId);
  this.photoOutPut = cameraManager.createPhotoOutput(photoProfile);
  this.setPhotoOutputCb(this.photoOutPut);

  this.photoSession = cameraManager.createSession(camera.SceneMode.NORMAL_PHOTO) as camera.PhotoSession;
  this.photoSession.beginConfig();
  this.photoSession.addInput(this.cameraInput);
  this.photoSession.addOutput(this.previewOutput);
  this.photoSession.addOutput(this.photoOutPut);
  await this.photoSession.commitConfig();
  await this.photoSession.start();
}
```

**Ordering is the whole fix.** The shipped code opens the hardware camera and
*then* asks whether the device supports photo mode and whether the capability
object exists; on a device that fails either check the function returns with the
input open and the field about to be overwritten by the next call, so the camera
stays occupied until the process dies. Nothing in `open()` is needed by the
checks, so moving it down costs nothing.

The profile lookup is the second trap. Both `find()` calls compare against
hardcoded 1440x1080 constants; on a device that does not publish exactly that
profile they return `undefined`, and `createPreviewOutput(undefined, surfaceId)`
throws 7400101 inside an async function nobody catches. `?? profiles[0]` is the
minimal fallback.

**The capture and the flash — `entry/src/main/ets/component/PictureComponent.ets`** (as shipped)

```typescript
Image($r('app.media.capture'))
  .width(CameraConstants.CAPTURE_SIZE)
  .height(CameraConstants.CAPTURE_SIZE)
  .onClick(() => {
    this.isVisibleTimer = true;
    this.captureTimer = this.timerShooting;
    if (this.captureTimer !== 0) {
      this.isVisibleCapture = false;          // hide the control row during the countdown
    }
    setTimeout(() => {
      this.cameraUtils.capture(this.isFront);
      this.captureClickFlag = this.captureClickFlag + 1;
      this.isVisibleCapture = true;
    }, this.captureTimer * 1000);
  });
```

**`captureClickFlag` is a counter used as an event.** `FlashBlackComponent`
declares `@Consume @Watch('onCaptureClick') captureClickFlag: number` and runs
its animation whenever the number changes; incrementing rather than toggling
means two captures in a row still produce two flashes. That is the ArkUI way to
send a one-shot signal down a component tree that only shares state, and it is
worth copying wherever you would otherwise reach for a callback prop.

`isFront` becomes the `mirror` field of `PhotoCaptureSetting`, so a selfie is
stored the way the user saw it. The asset comes back through
`photoOutput.on('photoAssetAvailable')`, where
`MediaAssetChangeRequest.saveCameraPhoto()` plus `applyChanges` commits it to the
gallery and `photoAsset.getThumbnail()` fills the bottom-left preview.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.CAMERA",
    "reason": "$string:reason_camera",
    "usedScene": { "abilities": ["EntryAbility"] }
  }
]
```

`CAMERA` is `user_grant`, so `reason` must resolve to a real string resource.
`usedScene.when` is omitted and defaults to `inuse`, which is right here - the
sample releases the camera in `onPageHide`. No storage permission is declared
and none is needed: the photo reaches the gallery through
`MediaAssetChangeRequest.saveCameraPhoto()`, which writes through the media
library rather than the file system.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `deviceTypes` is `phone`, `tablet`, `2in1`, but the preview `XComponent` is a
  fixed 510 vp tall with a surface fixed at 1440x1920, so the layout does not
  adapt to a tablet or a resized 2in1 window. The profiles are likewise
  hardcoded 1440x1080 constants (`HW-18-0021`).
- The timer offers only 0, 3 and 7 seconds, driven by three drawables. Adding a
  value means adding an icon.
- There is no way to cancel a running countdown: the control row is hidden while
  it runs and the `setTimeout` handle is never stored.

## Pitfalls

- **`HW-18-0041`** (B/medium, probable): camera init races the XComponent
  surface - `aboutToAppear`'s grant callback and
  `TwiceReqPermissionButtonComponent` both call `cameraShooting` with
  `surfaceId` still `''`, because the XComponent that sets it is only built
  after `notHasPermission` flips. The doomed first init opens the camera and
  fails at `createPreviewOutput`, racing the real `onLoad` init whose un-awaited
  teardown can strand it. Fix: gate init on `surfaceId !== ''` and drive it from
  `onLoad` only. Systematic - the same shape appears in CameraTwist,
  PipwindowRecorder and CameraResolution.
- **`HW-18-0042`** (B/low, confirmed): early returns after `cameraInput.open()`
  leave the opened camera never closed. The scene-mode and capability checks sit
  after `open()` and plain-`return` on failure, and the field is overwritten by
  the next call, so the hardware camera stays orphaned and blocks other clients
  until app exit. Fix: reorder the checks before `open()`, or `close()` on each
  early return.
- **`HW-18-0010`** (B/medium, confirmed): systematic fire-and-forget camera
  teardown across eight samples. `releaseCamera()` never awaits `stop()`,
  `close()` or `release()` and never nulls the fields, and `cameraShooting`
  calls it without `await` before rebuilding - so the camera-switch button
  reconfigures while the old input is still closing, and `if (this.photoSession)`
  guards stay truthy on released objects. Fix: await each call in order, null the
  fields, await `releaseCamera()` in callers.
- **`HW-18-0021`** (B/medium, probable): systematic hardcoded camera profiles.
  Both `find()` lookups can return `undefined` and go straight into
  `createPreviewOutput` / `createPhotoOutput`; the `undefined` guard runs *after*
  the call that would have thrown. Fix: guard the results, fall back to the first
  supported profile.
- **`onPageShow` only re-checks when the permission was already granted** - the
  guard is `if (!this.notHasPermission)` and the flag starts `true`, so the
  refusal screen never recovers on resume. Invert it.
- **The countdown cannot be cancelled.** The `setTimeout` handle is discarded, so
  a 7-second timer runs to completion even if the user leaves the page.

## References

- `documentation/harmonyos-guides/05_media/camera-device-management.md` - device query, `getSupportedOutputCapability`, profile selection
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-device-management
- `documentation/harmonyos-guides/05_media/camera-session-management.md` - `beginConfig` / `commitConfig` / `start` and teardown order
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-session-management
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - `XComponentController`, `getXComponentSurfaceId`, `onLoad`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-texttimer.md` - `isCountDown`, `count`, `format`, `TextTimerController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-texttimer
- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` - `setTimeout`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-timer
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - `requestPermissionsFromUser`, `requestPermissionOnSetting`, `checkAccessTokenSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `ohos.permission.CAMERA`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `documentation/harmonyos-references/04_media/arkts-apis-photoAccessHelper-PhotoAccessHelper.md` - `MediaAssetChangeRequest`, `saveCameraPhoto`, `applyChanges`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `PHOTO-25` (AVRecorderTimer) - the same timer affordance applied to recording
