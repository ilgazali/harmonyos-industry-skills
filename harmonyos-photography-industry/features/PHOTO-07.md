---
id: PHOTO-07
title: Custom camera zoom - pinch and preset buttons clamped to the session's own zoom range
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/07_customised_camera.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/customised_camera-0000002279772340
sample: huawei_industry_tree/18_photography/downloads/CustomisedCamera.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CameraKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [abilityAccessCtrl, camera, common, curves, display, hilog, image, photoAccessHelper, window, PinchGesture, onActionUpdate, onActionEnd, XComponent, XComponentController, getZoomRatioRange, setZoomRatio, getZoomRatio, PhotoCaptureSetting, photoAssetAvailable, MediaAssetChangeRequest, saveCameraPhoto, animateToImmediately, "curves.interpolatingSpring", "@LocalStorageLink", "@StorageLink"]
permissions: [ohos.permission.CAMERA]
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0033, HW-18-0010, HW-18-0027, HW-18-0030, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when you are building **your own camera UI** - not
`cameraPicker` - and the user must be able to zoom. That is any in-app camera
with a viewfinder of its own: document capture, a scanner, a social camera, a
measuring or inspection tool.

The pattern is small and exact. The session, once committed, tells you what it
can do: `photoSession.getZoomRatioRange()` returns `[min, max]`. A
`PinchGesture` multiplies the *ratio at the start of the gesture* by
`event.scale`, clamps the product to that range, and pushes it in with
`setZoomRatio`. On `onActionEnd` the effective ratio is read back with
`getZoomRatio` and becomes the base for the next pinch. Four preset buttons
(1x, 2x, 4x, 10x) write the same state through the same setter, so gesture and
buttons cannot disagree.

It generalises to every continuous camera control with a device-reported
range: exposure bias, focus distance, aperture. The shape is always
*ask the session for the range, clamp, set, read back*. Never trust a
hardcoded maximum - the range differs per device, per lens and per scene mode.

This sample also carries a clean shutter-flash animation and the modern
deferred-photo save path (`photoAssetAvailable` +
`MediaAssetChangeRequest.saveCameraPhoto`), both worth lifting on their own.

## Feature checklist

- Full-screen black camera page: flash-mode button, live preview, zoom UI,
  bottom row with thumbnail, shutter and camera-switch.
- Two-finger pinch on the preview changes the zoom continuously and shows the
  current ratio as an overlay (`wide angle` at the minimum, `3.2x` otherwise).
- The zoom is clamped to `getZoomRatioRange()` at both ends.
- Releasing the pinch reads the ratio actually applied by the session and
  keeps it for the next gesture and the next page entry.
- A 1x / 2x / 4x / 10x preset bar highlights whichever band the current ratio
  falls into and jumps to that ratio when tapped.
- The flash button cycles off / on / auto / always-on with a matching icon.
- The shutter triggers a capture and a black flash-through animation over the
  preview.
- Captured photos land in the gallery; the last one appears as a round
  thumbnail in the bottom left.
- Tapping the thumbnail opens the photo in the system gallery.
- Switching to the front camera resets the zoom to 1x and hides the zoom UI and
  flash controls.

## Architecture

One `entry` module. A camera singleton, one page, two small components.

```
entry/src/main/ets
├── component/
│   ├── FlashBlackComponent.ets      the shutter flash-through, driven by @Watch on a counter
│   └── FlashingLightComponent.ets   the four-state flash-mode cycler
├── constants/CameraConstants.ets    profiles (1440x1080), sizes, the 2/4/10 zoom thresholds
├── entryability/EntryAbility.ets    full screen, avoid areas -> AppStorage, context handoff
├── entrybackupability/EntryBackupAbility.ets
├── pages/MainPage.ets               @Entry: preview, pinch, preset bar, action row
└── utils/
    ├── CameraUtils.ets              the singleton: session, input, preview/photo outputs, save
    └── Logger.ets
```

The documented 工程目录 matches the zip exactly.

**The design decision worth copying** is that zoom state is *never* stored in
the component and read back from it. `CameraUtils.setPhotoZoom(z)` is a
one-line wrapper over `photoSession.setZoomRatio(z)`, and every path - pinch,
each of the four presets, the camera switch - goes through it. The `@LocalStorageLink('zoom')`
in the page exists only so the ratio text and the preset highlighting can
re-render. The session is the source of truth, and `onActionEnd` re-reads it
with `getPhotoZoom()` precisely because the hardware may have clamped or
quantised the value the app asked for.

**The decision worth avoiding** is the three module-level `let`s at the top of
`MainPage.ets` - `cameraPosition`, `surfaceId`, `zoomRatioRange`, `currentFov`,
`savedZoom`. They are file-scoped globals used to survive page re-entry.
It works because there is exactly one instance of the page, and it will break
the day there are two (a split-screen 2in1 window, a second camera tab). The
same values belong in `AppStorage` alongside `photoUri`, which the same file
already uses correctly.

## Implementation steps

1. **Declare only `ohos.permission.CAMERA`.** The photo save path here is the
   deferred `photoAssetAvailable` flow, which needs no album permission.
2. **Request it and check the result** before touching the camera manager
   (`HW-18-0030`) - the sample requests and then builds the pipeline in a bare
   `.then()` regardless of the answer.
3. **Take the preview surface from the `XComponent`** in `onAttach`, after
   `setXComponentSurfaceRect` has fixed its size.
4. **Build the session**: `getSupportedCameras`, check
   `SceneMode.NORMAL_PHOTO` is supported, `find` a preview and a photo profile
   by exact width/height/format, `createPreviewOutput` + `createPhotoOutput`,
   `beginConfig` / `addInput` / `addOutput` x2 / `commitConfig` / `start`.
5. **Read the capabilities *after* `start()`**: `hasFlash()`,
   `isFocusModeSupported()`, and `getZoomRatioRange()`. Return the range to the
   page - it is the clamp for every later zoom write.
6. **Clamp inside `onActionUpdate`**, not in the setter: `currentFov * event.scale`,
   then `> range[1]` -> `range[1]`, `< range[0]` -> `range[0]`.
7. **Re-read the applied ratio in `onActionEnd`** with `getZoomRatio()` and
   store it as the base for the next gesture.
8. **Feed the device orientation into `PhotoCaptureSetting.rotation`**
   (`HW-18-0027`) - the sample hardcodes `ROTATION_0`, so photos taken sideways
   are saved sideways.
9. **Await the teardown before rebuilding** (`HW-18-0010`): `releaseCamera()`
   here fires five native calls without awaiting any of them, and the camera
   switch calls it immediately before `cameraShooting()` rebuilds the session.
10. **Open the gallery with an implicit want** (`HW-18-0033`), not a hardcoded
    system bundle name.

## Verified snippets

All snippets are from `CustomisedCamera.zip`. Corrected forms are marked.

**Pinch to zoom - `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
let zoomRatioRange: number[] = [];
let currentFov: number = 1;
let savedZoom: number = 1;                       // 保存当前焦距

XComponent({
  type: XComponentType.SURFACE,
  controller: this.mXComponentController,
  imageAIOptions: {
    types: [ImageAnalyzerType.SUBJECT],
    aiController: this.aiController
  }
})
  .gesture(
    PinchGesture({ fingers: CameraConstants.FINGER_TWO })
      .onActionUpdate((event: GestureEvent) => {
        if (event) {
          this.zoom = currentFov * event.scale;   // scale is relative to the gesture start
          this.isShowZoom = true;
          if (this.zoom > (zoomRatioRange[1])) {
            this.zoom = zoomRatioRange[1];
          } else if (this.zoom < zoomRatioRange[0]) {
            this.zoom = zoomRatioRange[0];
          }
          CameraUtils.setPhotoZoom(this.zoom);
        }
      })
      .onActionEnd(() => {
        currentFov = CameraUtils.getPhotoZoom();  // what the session actually applied
        savedZoom = currentFov;
        this.isShowZoom = false;
      })
  )
  .onAttach(async () => {
    this.mXComponentController.setXComponentSurfaceRect({
      surfaceWidth: this.imageSize.width,
      surfaceHeight: this.imageSize.height
    });
    surfaceId = this.mXComponentController.getXComponentSurfaceId();
  })
  .hitTestBehavior(HitTestMode.Block);
```

**Three things carry the design.** `event.scale` in `PinchGesture` is
*cumulative within one gesture* and resets to 1 at the next touch-down, which
is why the base has to be a separate variable (`currentFov`) rather than the
previous frame's zoom - multiplying the live value would compound the scale and
the preview would run away. `hitTestBehavior(HitTestMode.Block)` stops the
pinch leaking to ancestors, which matters because the preview sits in a `Stack`
under the zoom bar. And `onActionEnd` reads back through `getPhotoZoom()`
rather than trusting `this.zoom`: `setZoomRatio` is a request, and a device
with a 1x-10x optical range will quantise or clamp it.

The four preset buttons write the same three variables and call the same
setter, and their highlight condition is a band test - `zoom < 2`,
`2 <= zoom < 4`, `4 <= zoom < 10`, `zoom >= 10` - so a pinch to 3.2x lights the
2x button. That is the right behaviour for a preset bar and it comes for free
because both inputs converge on one number.

**Session setup and where the range comes from - `entry/src/main/ets/utils/CameraUtils.ets`** (as shipped, trimmed)

```typescript
async cameraShooting(cameraPosition: number, surfaceId: string, context: Context): Promise<number[]> {
  this.releaseCamera();                          // not awaited - see HW-18-0010

  let cameraManager: camera.CameraManager = camera.getCameraManager(context);
  let cameraArray: camera.CameraDevice[] = cameraManager.getSupportedCameras();
  if (cameraArray.length <= 0) {
    return [];
  }

  this.cameraInput = cameraManager.createCameraInput(cameraArray[cameraPosition]);
  await this.cameraInput.open();

  let sceneModes: camera.SceneMode[] = cameraManager.getSupportedSceneModes(cameraArray[cameraPosition]);
  let cameraOutputCap: camera.CameraOutputCapability =
    cameraManager.getSupportedOutputCapability(cameraArray[cameraPosition], camera.SceneMode.NORMAL_PHOTO);
  if (sceneModes.indexOf(camera.SceneMode.NORMAL_PHOTO) < 0) {
    return [];
  }

  let previewProfile: undefined | camera.Profile = cameraOutputCap.previewProfiles.find(
    (profile: camera.Profile) => {
      return profile.size.width === this.previewProfileObj.size.width &&
        profile.size.height === this.previewProfileObj.size.height &&
        profile.format === this.previewProfileObj.format;
    });

  this.previewOutput = cameraManager.createPreviewOutput(previewProfile, surfaceId);
  this.photoOutPut = cameraManager.createPhotoOutput(photoProfile);
  this.setPhotoOutputCb(this.photoOutPut);       // 分段式保存图片

  this.photoSession = cameraManager.createSession(camera.SceneMode.NORMAL_PHOTO) as camera.PhotoSession;
  this.photoSession.beginConfig();
  this.photoSession.addInput(this.cameraInput);
  this.photoSession.addOutput(this.previewOutput);
  this.photoSession.addOutput(this.photoOutPut);
  await this.photoSession.commitConfig();
  await this.photoSession.start();

  if (this.photoSession.hasFlash()) {
    this.photoSession.setFlashMode(camera.FlashMode.FLASH_MODE_CLOSE);
  }
  if (this.photoSession.isFocusModeSupported(camera.FocusMode.FOCUS_MODE_CONTINUOUS_AUTO)) {
    this.photoSession.setFocusMode(camera.FocusMode.FOCUS_MODE_CONTINUOUS_AUTO);
  }
  let zoomRatioRange = this.photoSession.getZoomRatioRange();
  return zoomRatioRange;                         // the page's clamp
}
```

**The capability queries are all after `commitConfig`/`start`, and that is
required.** `hasFlash`, `isFocusModeSupported` and `getZoomRatioRange` describe
*this configured session on this device with this lens*, not the camera in the
abstract - a 3x telephoto and an ultra-wide on the same phone report different
ranges, and switching scene mode changes them again. Returning the range from
the setup function rather than exposing a getter is what guarantees the page
cannot clamp against a stale range: the only way to get one is to have just
built a session.

Note the profile selection matches on **format as well as size**
(`PHOTOPROFILE_FORMAT: 2000`, `PREVIEWPROFILE_FORMAT: 1003`). If `find`
returns `undefined` - very possible on another device - `createPreviewOutput`
is still called with it and the session fails at commit, silently.

**Teardown and capture rotation - same file** (corrected, see `HW-18-0010`, `HW-18-0027`)

```typescript
async releaseCamera(): Promise<void> {
  if (this.photoSession) {
    await this.photoSession.stop();              // FIX: sample fires all five without await
  }
  if (this.cameraInput) {
    await this.cameraInput.close();
  }
  if (this.previewOutput) {
    await this.previewOutput.release();
  }
  if (this.photoOutPut) {
    await this.photoOutPut.release();
  }
  if (this.photoSession) {
    await this.photoSession.release();
  }
  // FIX: null the fields, otherwise the `if (this.photoSession)` guards above
  //      stay truthy on released objects the next time round.
  this.photoSession = undefined;
  this.cameraInput = undefined;
  this.previewOutput = undefined;
  this.photoOutPut = undefined;
}

capture(isFront: boolean, rotation: camera.ImageRotation): void {
  let settings: camera.PhotoCaptureSetting = {
    quality: camera.QualityLevel.QUALITY_LEVEL_HIGH,
    rotation: rotation,                          // FIX: sample hardcodes ImageRotation.ROTATION_0
    mirror: isFront
  };
  if (this.photoOutPut) {
    this.photoOutPut.capture(settings);
  }
}
```

**Camera teardown is asynchronous whether or not you wait for it.** The switch
handler in `MainPage` calls `CameraUtils.cameraShooting(...)` which begins with
an un-awaited `this.releaseCamera()`, so `createCameraInput` and `open()` for
the new lens run while the old input is still closing. The visible symptoms are
intermittent: a camera-occupied error, a black preview after a fast double-tap
on the switch button, and unhandled rejections from the five dropped promises.
The same shape occurs in nine samples across this industry (`HW-18-0010`);
`VideoRecording` (`PHOTO-06`) is the one that awaits correctly and is the
reference to copy.

`mirror: isFront` is right - a front-camera photo should match the preview the
user was looking at. `rotation: ROTATION_0` is not: `RatioCamera` in this same
industry even computes the sensor rotation from the gravity sensor and then
ignores it at capture time (`HW-18-0027`). A photo taken with the phone
sideways is written sideways and shows sideways in the gallery.

**Deferred save and the gallery hand-off - same file** (corrected, see `HW-18-0033`)

```typescript
setPhotoOutputCb(photoOutput: camera.PhotoOutput): void {
  photoOutput.on('photoAssetAvailable',
    async (_err: BusinessError, photoAsset: photoAccessHelper.PhotoAsset): Promise<void> => {
      let accessHelper: photoAccessHelper.PhotoAccessHelper =
        photoAccessHelper.getPhotoAccessHelper(CameraUtils.currentContext);
      let assetChangeRequest: photoAccessHelper.MediaAssetChangeRequest =
        new photoAccessHelper.MediaAssetChangeRequest(photoAsset);
      assetChangeRequest.saveCameraPhoto();
      await accessHelper.applyChanges(assetChangeRequest);
      this.uri = photoAsset.uri;
      AppStorage.setOrCreate('photoUri', await photoAsset.getThumbnail());
    });
}

// 跳转系统相册
previewPhoto(context: Context): void {
  let photoContext = context as common.UIAbilityContext;
  photoContext.startAbility({
    action: 'ohos.want.action.viewData',         // FIX: sample also pins
    uri: this.uri,                               //   bundleName: 'com.huawei.hmos.photos'
    type: 'image/*'                              //   abilityName: '...photos.MainAbility'
  });
}
```

**`photoAssetAvailable` is the low-latency capture path and it needs no album
permission.** The camera framework creates the asset itself and hands the app a
`PhotoAsset`; `saveCameraPhoto()` on a `MediaAssetChangeRequest` commits it,
and `getThumbnail()` gives a `PixelMap` for the round preview in the corner
without ever decoding the full image. This is why `module.json5` here declares
only `CAMERA` and none of the restricted `READ`/`WRITE_IMAGEVIDEO` that nine
sibling samples declare unnecessarily (`HW-18-0004`) - and it is the pattern
those samples should have used.

The gallery hand-off is the one weak spot. Pinning `bundleName`/`abilityName`
to `com.huawei.hmos.photos` is not a public contract: on a device where the
gallery has a different bundle, or after an interface change, `startAbility`
rejects and the thumbnail tap does nothing at all - no error, no toast. An
implicit want with `action` and `uri` lets the system resolve whatever image
viewer is installed.

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

- `CAMERA` is `user_grant`, so `reason` and `usedScene` are mandatory.
- `usedScene` omits `when`, which defaults to `inuse` - correct for a
  foreground camera. Compare `PHOTO-06`, which claims `always` for no reason.
- **No album permission at all**, and none is needed: the save runs through
  `photoAssetAvailable` + `saveCameraPhoto`. This is the sample to point at
  when removing the restricted permissions from the nine that declare them
  (`HW-18-0004`).
- No microphone: this is a stills camera.
- `deviceTypes` is `phone`, `tablet`, `2in1`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **The profiles are hardcoded to 1440x1080** with fixed format codes for both
  preview and photo. On a device that does not expose that exact triple, `find`
  yields `undefined`, the outputs are created with it anyway, and the session
  fails at `commitConfig` with a black preview and only a log line.
- The preview `XComponent` surface is set to 1440x1920 while the preview
  profile is 1440x1080 - a 4:3 stream in a 3:4 box. The preview is stretched.
- `zoomRatioRange` is a module-level `let` that stays `[]` until
  `cameraShooting` resolves. A pinch in that window compares against
  `undefined` and both clamps fail; `this.zoom` then goes wherever the gesture
  takes it. Guard on `zoomRatioRange.length === 2`.
- Zoom is a `PhotoSession` capability here. In `VideoSession` (`PHOTO-06`) the
  API is the same but the ranges differ, so do not cache one for the other.
- The front camera hides the flash button, the zoom bar and the ratio overlay
  (`visibility(this.isFront ? Hidden : Visible)`), so pinch still works on the
  front camera but shows no feedback.
- `FlashBlackComponent` drives its animation from `@Link @Watch` on an
  incrementing counter rather than a boolean, which is what lets two captures
  in quick succession each get their own flash.
- `EntryAbility.onWindowStageCreate` reads the avoid areas once and never
  registers `avoidAreaChange`, so a rotation or a foldable unfold leaves stale
  padding.

## Pitfalls

- **`HW-18-0033`** (B/low, confirmed): `previewPhoto` launches the gallery with
  a hardcoded internal bundle and ability name (`com.huawei.hmos.photos` /
  `...photos.MainAbility`) instead of a public action. It is not an API
  contract, it breaks wherever the bundle differs, and the failure is silent -
  the thumbnail tap does nothing. Fix: an implicit want carrying only
  `action: 'ohos.want.action.viewData'` and the `uri`, or the media library's
  own preview capability.
- **`HW-18-0010`** (B/medium, confirmed): systematic - fire-and-forget camera
  teardown races the rebuild in eight photography samples, this one at
  `CameraUtils.ets:57` (the un-awaited `releaseCamera()` at the head of
  `cameraShooting`) and `:170-186` (the five un-awaited native calls). None of
  them null the fields, so `if (this.photoSession)` stays truthy on a released
  session. Symptoms: intermittent camera-occupied errors, black preview after a
  fast camera switch, unhandled rejections. Fix: await each call in order
  (stop -> close -> release), reset the fields, and await `releaseCamera()`
  before re-initialising.
- **`HW-18-0027`** (B/low, confirmed): systematic - `capture()` hardcodes
  `camera.ImageRotation.ROTATION_0` in three samples (`CustomisedCamera`
  `CameraUtils.ets:153-162`, `RatioCamera`, `CameraResolution`). `RatioCamera`
  even computes the sensor rotation and uses it only for the thumbnail UI.
  Photos taken with the device rotated are saved with the wrong orientation.
  Fix: derive the rotation from the device orientation and pass it into
  `PhotoCaptureSetting`.
- **`HW-18-0030`** (B/medium, confirmed): systematic - `MainPage.onPageShow`
  (`:46-58`) calls `requestPermissionsFromUser` and then builds the whole
  camera pipeline in the `.then()` without reading `authResults`, so denying
  the dialog produces raw camera errors instead of a refused-state UI. This
  sample does at least have a `.catch` on the request, unlike the five others
  in the group. Fix: check `data.authResults` before building.
- **Uninitialised clamp**: `zoomRatioRange` is empty until the session is up;
  a pinch before that is unclamped.

## References

- `huawei_industry_tree/18_photography/docs/07_customised_camera.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customised_camera-0000002279772340
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pinchgesture.md` - `PinchGesture`, `event.scale`, `onActionUpdate` / `onActionEnd`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pinchgesture
- `documentation/harmonyos-references/04_media/js-apis-camera.md` - `PhotoSession`, `getZoomRatioRange`, `setZoomRatio`, `PhotoCaptureSetting`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-camera
- `documentation/harmonyos-references/04_media/arkts-apis-camera-cameramanager.md` - `getSupportedCameras`, `getSupportedOutputCapability`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-camera-cameramanager
- `documentation/harmonyos-guides/05_media/camera-shooting-case.md` - the full stills pipeline
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-shooting-case
- `documentation/harmonyos-guides/05_media/camera-session-management.md` - the `beginConfig`/`commitConfig` transaction and teardown order
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-session-management
- `documentation/harmonyos-guides/05_media/camera-rotation.md` - deriving the capture rotation from device orientation
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-rotation
- `documentation/harmonyos-guides/05_media/camera-deferred-capture.md` - `photoAssetAvailable` and `saveCameraPhoto`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-deferred-capture
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - the preview surface
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - `ohos.permission.CAMERA`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `PHOTO-06` - the video sibling, and the one sample whose `releaseCamera` awaits correctly
- `PHOTO-04` - the aspect-ratio camera, which shares the hardcoded capture rotation
