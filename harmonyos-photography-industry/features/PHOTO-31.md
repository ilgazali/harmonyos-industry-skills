---
id: PHOTO-31
title: Preview resolution switch - pick a high or low preview profile by aspect ratio and rebuild the session (doc/sample mismatch)
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/31_audio-v1_2-ts_50.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/audio-v1_2-ts_50-0000002407536358
sample: huawei_industry_tree/18_photography/downloads/CameraResolution.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkGraphics2D", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CameraKit", "@kit.CoreFileKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit", "@kit.PreviewKit"]
apis: [abilityAccessCtrl, camera, colorSpaceManager, common, curves, dataSharePredicates, display, filePreview, hilog, photoAccessHelper, window]
permissions: [ohos.permission.CAMERA, ohos.permission.WRITE_IMAGEVIDEO, ohos.permission.READ_IMAGEVIDEO]
min_api: 20
modules: [entry]
findings: [HW-18-0089, HW-18-0008, HW-18-0010, HW-18-0021, HW-18-0023, HW-18-0027, HW-18-0030, HW-18-0041, HW-18-0092]
status: verified-with-fixes
---

## When to use

**Read this section before anything else: the document and the sample are not
the same feature.**

`huawei_industry_tree/18_photography/docs/31_audio-v1_2-ts_50.md` is titled
使用OpenGL标准化设备坐标绘制矩形人脸框 - drawing a face rectangle with OpenGL ES
normalised device coordinates - and describes a project with a `cpp` directory,
`faceDetector`, and an `ArrayBuffer` round-trip back to ArkTS. The zip carried
under this feature id is `CameraResolution.zip`, which contains no native code,
no `faceDetector`, and no OpenGL: it is a camera page that switches the preview
between a low and a high resolution profile. That sample belongs to
`26_insurance-v1_2-ts_32.md` (自定义相机分辨率设置), which in this repo is a
doc-only card, `PHOTO-26`. Both documents are also published under slugs naming
unrelated industries - `audio-…` and `insurance-…` - one of six such instances
(`HW-18-0008`).

**Everything verified below is `CameraResolution`.** Nothing in this card is
evidence about the OpenGL face-box sample; that source was never shipped with
this page. For the face-box technique, use the document itself and treat its
snippets as unverified.

So: load this card when **the user must be able to change preview quality at
runtime** - a resolution toggle, an HD switch, a data-saver mode. The pattern is
that a resolution change is not a setting you apply, it is a session you
rebuild: profiles are fixed at `createPreviewOutput` time, so switching means
tearing the whole pipeline down and configuring a new one against the same
surface. Everything else in the page - zoom, the shutter, the thumbnail - has
to survive that.

## Feature checklist

- The page opens on a live preview in a fixed-ratio `XComponent` with the
  surface rotation locked.
- A toolbar toggle switches between the lowest and the highest preview profile
  that matches the target aspect ratio; the toggle's icon reflects the state and
  reacts on touch-down.
- Switching rebuilds the camera session in place: the preview stays on the same
  surface and the same size, only the resolution changes.
- The shutter captures at `QUALITY_LEVEL_HIGH` in high mode and
  `QUALITY_LEVEL_LOW` in low mode.
- Photos are saved to the gallery through `saveCameraPhoto()` and the newest
  thumbnail appears in the corner.
- Tapping the thumbnail opens the photo in the system gallery; returning to the
  page closes the preview overlay.
- Pinch-to-zoom scales the preview between the session's zoom bounds.
- Backgrounding the app releases the camera; returning rebuilds it.

## Architecture

One `entry` module, four source files. All camera state is module-level in
`CameraShooter.ets`; all page state is module-level in `Index.ets`.

```
entry/src/main/ets
├── constants/CameraConstants.ets   sizes, SURFACE_RATIO 632/400, EPSILON 0.03
├── entryability/EntryAbility.ets   full screen; onForeground -> fromBack, onBackground -> releaseCamera
├── entrybackupability/EntryBackupAbility.ets
├── pages/Index.ets                 @Entry: toolbar, XComponent, control panel; exports fromBack()
└── utils/CameraShooter.ets         session build/teardown, profile selection, capture, save, preview
```

The documented tree does not match the zip at all - it lists
`entry/src/main/cpp/face_box.cpp` and a `CMakeLists.txt` that do not exist here,
because it is a different project's tree (`HW-18-0008`).

**The design decision worth copying** is how the two profiles are chosen. The
sample does not hardcode sizes. It walks `cameraOutputCap.previewProfiles`
forward to find the *first* profile whose aspect ratio matches the target, and
backward to find the *last* one - relying on the platform returning the array
sorted by size. Low mode gets the first, high mode gets the last, and both are
guaranteed to exist on the device because they came out of its own capability
list. The ratio comparison uses an epsilon (`0.03`) rather than equality, which
is what makes it survive profiles like 1920x1088 that are only nominally 16:9.

**The decision worth avoiding** is the ownership of the rebuild. Three
independent things start the camera: `aboutToAppear`'s `setTimeout(..., 200)`,
the toolbar's `onTouch`, and `EntryAbility.onForeground` → `fromBack()`. None of
them awaits `releaseCamera()`, none checks whether an init is already running,
and `onForeground` fires on first launch *and* every time the permission dialog
dismisses (`HW-18-0089`). The page needs one owner and one in-flight guard;
`fromBack` should ask whether there is a surface and a grant before it rebuilds
anything.

## Implementation steps

1. **Take the surface id in `XComponent.onLoad`,** after
   `setXComponentSurfaceRotation({ lock: true })` and
   `setXComponentSurfaceRect(...)`. The locked rotation is what lets the preview
   ignore device turns (`HW-18-0041`).
2. **Request `CAMERA` and read `authResults`** before any camera call
   (`HW-18-0030`).
3. **Ask the device for its profiles.** `getSupportedOutputCapability(camera,
   SceneMode.NORMAL_PHOTO)` gives `previewProfiles` and `photoProfiles`; never
   hardcode a size (`HW-18-0021` is the sibling samples doing exactly that).
4. **Match on aspect ratio with an epsilon,** not on exact width and height.
5. **Pick first-match for low and last-match for high,** and bail out with a log
   if either is undefined.
6. **Pick the photo profile from the reversed `photoProfiles`** so the largest
   matching still image is used regardless of the preview mode.
7. **Set the colour space before `commitConfig()`:**
   `photoSession.setColorSpace(colorSpaceManager.ColorSpace.DISPLAY_P3)`.
8. **Await the teardown before rebuilding.** A resolution switch is a full
   session rebuild and the old input must be closed first (`HW-18-0010`).
9. **Gate the foreground rebuild** on grant, non-empty surface id, and no init
   in flight (`HW-18-0089`).
10. **Assign the returned `zoomRatioRange`** at every call site, or the pinch
    clamp compares against `undefined` (`HW-18-0023`).
11. **Derive the capture rotation from the device** rather than hardcoding
    `ROTATION_0` (`HW-18-0027`; see `PHOTO-29` for the sensor mapping).

## Verified snippets

All snippets are from `CameraResolution.zip`. Corrected forms are marked.

**Profile selection — `entry/src/main/ets/utils/CameraShooter.ets`** (as shipped)

```typescript
let photoSize: Size = { width: 1920, height: 1080 };   // target ratio only, not a requested size

export async function cameraShooting(highResolution: boolean, surfaceId: string, context: Context): Promise<number[]> {
  let zoomRatioRange: number[] = [];
  currentContext = context;
  resolution = highResolution;
  releaseCamera();
  try {
    let cameraManager: camera.CameraManager = camera.getCameraManager(context);
    let cameraArray: camera.CameraDevice[] = cameraManager.getSupportedCameras();
    cameraInput = cameraManager.createCameraInput(cameraArray[0]);
    await cameraInput.open();
    let cameraOutputCap: camera.CameraOutputCapability =
      cameraManager.getSupportedOutputCapability(cameraArray[0], camera.SceneMode.NORMAL_PHOTO);
    let previewProfilesArray: camera.Profile[] = cameraOutputCap.previewProfiles;

    let lowResolutionProfile: camera.Profile | undefined = undefined;
    let highResolutionProfile: camera.Profile | undefined = undefined;
    for (let i = 0; i < previewProfilesArray.length; i++) {          // forward: smallest match
      if (Math.abs(previewProfilesArray[i].size.width / previewProfilesArray[i].size.height -
        photoSize.width / photoSize.height) < CameraConstants.EPSILON) {
        lowResolutionProfile = previewProfilesArray[i];
        break;
      }
    }
    for (let i = previewProfilesArray.length - 1; i >= 0; i--) {     // backward: largest match
      if (Math.abs(previewProfilesArray[i].size.width / previewProfilesArray[i].size.height -
        photoSize.width / photoSize.height) < CameraConstants.EPSILON) {
        highResolutionProfile = previewProfilesArray[i];
        break;
      }
    }
    if (lowResolutionProfile === undefined || highResolutionProfile === undefined) {
      hilog.error(DOMAIN, TAG, 'previewProfiles not found!');
      return [];
    }
    let previewProfile = highResolution ? highResolutionProfile : lowResolutionProfile;
    previewOutput = cameraManager.createPreviewOutput(previewProfile, surfaceId);
```

**`photoSize` is a ratio, not a resolution.** It is never used as a size - only
as `photoSize.width / photoSize.height`, i.e. 16:9. Renaming it `TARGET_RATIO`
would make the whole function legible. `EPSILON = 0.03` is the tolerance that
lets 1920x1088 (ratio 1.7647) match 16:9 (1.7778); an equality test would reject
it and the feature would silently have no profiles on many devices.

The two loops are the same predicate walked in opposite directions, which is the
cheapest possible way to get "smallest matching" and "largest matching" out of a
sorted capability list. The `undefined` check sits **before** the create call
here - unlike four sibling samples where the equivalent guard sits after it, or
re-tests the wrong variable (`HW-18-0021`; this file's own photo-profile guard at
line 109 re-tests `previewOutput === undefined` where it means `photoProfile`).

Note also `releaseCamera()` on the third line, un-awaited, immediately before the
new pipeline is built (`HW-18-0010`).

**Capture quality follows the mode — same file** (as shipped)

```typescript
export async function capture() {
  try {
    let settings: camera.PhotoCaptureSetting = {
      // 根据是否开启高分辨率模式选取quality  (choose the quality by the high-resolution mode)
      quality: resolution ? camera.QualityLevel.QUALITY_LEVEL_HIGH : camera.QualityLevel.QUALITY_LEVEL_LOW,
      rotation: camera.ImageRotation.ROTATION_0,       // see HW-18-0027 - should follow the device
      mirror: false
    };
    photoOutPut.capture(settings);
  } catch (error) {
    hilog.error(DOMAIN, TAG, `The capture call failed. error: ${error.code}`);
  }
}
```

**`resolution` is a module-level boolean written by `cameraShooting`**, so the
still-image quality automatically tracks whatever the preview toggle last
selected. That coupling is the point of the sample: one user-facing switch moves
both the preview profile and the encode quality, and the capture path never has
to know about the UI.

`rotation: ROTATION_0` is hardcoded, so any photo taken with the device turned is
saved sideways in the gallery - the same defect as two sibling samples
(`HW-18-0027`). This sample has no sensor code at all to fix it with; take the
gravity mapping from `PHOTO-29` or the `videoOutput.getVideoRotation` approach
from `PHOTO-30`.

**Session config and colour space — same file** (as shipped)

```typescript
    let photoProfilesArray: camera.Profile[] = cameraOutputCap.photoProfiles.slice().reverse();
    let photoProfile: camera.Profile | undefined = undefined;
    if (previewProfile !== undefined) {
      photoProfile = photoProfilesArray.find((profile: camera.Profile) => {
        return Math.abs(profile.size.width / profile.size.height - photoSize.width / photoSize.height) <
        CameraConstants.EPSILON;
      });
    }
    photoOutPut = cameraManager.createPhotoOutput(photoProfile);
    setPhotoOutputCb(photoOutPut);

    photoSession = cameraManager.createSession(camera.SceneMode.NORMAL_PHOTO) as camera.PhotoSession;
    photoSession.beginConfig();
    photoSession.addInput(cameraInput);
    photoSession.addOutput(previewOutput);
    photoSession.addOutput(photoOutPut);
    photoSession.setColorSpace(colorSpaceManager.ColorSpace.DISPLAY_P3);
    await photoSession.commitConfig();
    await photoSession.start();
    zoomRatioRange = photoSession.getZoomRatioRange();
```

**The order is not negotiable.** `beginConfig` → `addInput` → `addOutput` ×n →
`setColorSpace` → `commitConfig` → `start`: the colour space is part of the
configuration, so setting it after `commitConfig` has no effect, and
`getZoomRatioRange()` returns nothing meaningful until the session is started.
`DISPLAY_P3` is a deliberate choice for a photography app - a wider gamut than
sRGB, at the cost of needing a P3-capable display to show the difference.

`photoProfiles.slice().reverse()` copies before reversing, which matters: the
array belongs to the capability object and reversing it in place would corrupt
the next lookup. The reversed order plus `find()` yields the *largest* matching
photo profile, so still images stay full-size even in low-preview mode.

**Foreground rebuild, gated — `entry/src/main/ets/entryability/EntryAbility.ets` and `pages/Index.ets`** (corrected, see `HW-18-0089`, `HW-18-0041`, `HW-18-0030`)

```typescript
// EntryAbility.ets
onForeground(): void {
  fromBack(this.context);            // shipped: unconditional
  hilog.info(0x0000, 'testTag', '%{public}s', 'Ability onForeground');
}

onBackground(): void {
  releaseCamera();
  hilog.info(0x0000, 'testTag', '%{public}s', 'Ability onBackground');
}

// Index.ets
let initInFlight: boolean = false;   // FIX: absent in the sample

export async function fromBack(context: Context): Promise<void> {
  if (!granted || surfaceId === '' || initInFlight) {   // FIX: none of these checks exist
    return;
  }
  initInFlight = true;
  try {
    storage.setOrCreate('zoom', 1);
    currentFov = 1;
    zoomRatioRange = await cameraShooting(highQuality, surfaceId, context);   // FIX: return value discarded
  } finally {
    initInFlight = false;
  }
}
```

**`onForeground` fires more often than "the user came back".** It fires on first
launch, before `CAMERA` is granted and before the `XComponent` has produced a
surface id - so `fromBack` calls `cameraShooting(highQuality, '', context)`,
which opens the camera input and then fails at `createPreviewOutput`. It fires
again when the permission dialog dismisses, at which point the sample's own
`setTimeout(..., 200)` init from `aboutToAppear` may still be running. Each
`cameraShooting` begins with an un-awaited `releaseCamera()`, so the two runs
tear down each other's half-built sessions: an intermittent black preview at
startup that no log explains (`HW-18-0089`, `HW-18-0041`, `HW-18-0010`).

Three guards fix it, and the third - an in-flight flag - is the one people
forget. Assigning the returned `zoomRatioRange` while you are there also repairs
the pinch clamp, which today compares `this.zoom` against `undefined` at both
ends and therefore never clamps (`HW-18-0023`).

`onBackground → releaseCamera()` is right and worth keeping: holding a camera
session in the background is refused on this platform. If you need it held, that
is `PHOTO-30`'s picture-in-picture window.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.CAMERA",           "reason": "$string:reason_camera",            "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.WRITE_IMAGEVIDEO", "reason": "$string:reason_write_imagevideo",  "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.READ_IMAGEVIDEO",  "reason": "$string:reason_read_imagevideo",   "usedScene": { "abilities": ["EntryAbility"] } }
]
```

- All three are requested at runtime, but only `CAMERA` is needed: the save path
  is `MediaAssetChangeRequest.saveCameraPhoto()`, which is permission-free.
- `READ_IMAGEVIDEO` / `WRITE_IMAGEVIDEO` are restricted (ACL) and cannot ship in
  an ordinary app - the same template inheritance as `HW-18-0004`.
- The document for this page declares no permissions at all, because it
  documents a different sample (`HW-18-0008`).
- `deviceTypes` is `phone` only, which is consistent: the surface height is
  derived from `display.getDefaultDisplaySync().width` at module load and never
  recomputed, so a resized window keeps the launch-time geometry.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **The shipped zip does not implement the document this card's `doc:` field
  points to.** Anything about `faceDetector`, OpenGL ES normalised device
  coordinates or the native `ArrayBuffer` round-trip is unverified
  (`HW-18-0008`).
- Only `cameraArray[0]` is ever opened; the front/back switch button's `onClick`
  is an empty comment (开发者可自定义摄像头切换逻辑: "developers can customise the
  camera switching logic").
- `getThumbnail()` exists on the page and is never called, so the corner preview
  only ever populates from the `photoAssetAvailable` callback.
- `filePreview.closePreview(this.context)` runs in `onPageShow` unconditionally,
  including the first show when nothing is open.
- `setPhotoSmoothZoom` is exposed but only reachable from `setZoom()`, which no
  handler calls; the pinch path uses the non-smooth `setZoomRatio`.

## Pitfalls

- **`HW-18-0089`** (B/medium, probable): `EntryAbility.onForeground` calls
  `fromBack` unconditionally, so the camera is rebuilt on first launch before
  the grant and before the surface exists, and again when the permission dialog
  dismisses - overlapping inits whose un-awaited teardowns destroy each other.
  Fix: gate on grant, surface id and an in-flight flag; serialise inits.
- **`HW-18-0008`** (E/low, confirmed, systematic): this page is published under
  the slug `audio-v1_2-ts_50` and describes the OpenGL face-box sample, while
  the zip attached here is `CameraResolution` - which belongs to
  `26_insurance-v1_2-ts_32`, itself another wrong-topic slug. Six such instances
  across the crawled industries. Fix: re-publish under topic-correct slugs and
  re-pair the samples.
- **`HW-18-0010`** (B/medium, confirmed, systematic): `releaseCamera()`
  (`CameraShooter.ets:181-203`) awaits nothing and nulls nothing, and
  `cameraShooting` calls it un-awaited at line 40 immediately before rebuilding.
  Since the resolution toggle rebuilds on every press, this sample hits the race
  more often than most. Fix: await `stop` → `close` → `release` and reset the
  fields.
- **`HW-18-0021`** (B/medium, probable, systematic): the photo-profile guard at
  `CameraShooter.ets:109` re-tests `previewOutput === undefined` where it means
  `photoProfile`, so `createPhotoOutput(undefined)` is reachable. Fix: guard the
  variable you just computed.
- **`HW-18-0023`** (B/low, confirmed, systematic): `zoomRatioRange` is never
  assigned - `Index.ets:36` declares it and every `cameraShooting` call site
  discards the return value - so both pinch clamps compare against `undefined`
  and never fire. Fix: assign the returned range.
- **`HW-18-0027`** (B/low, confirmed, systematic): `capture()` hardcodes
  `rotation: camera.ImageRotation.ROTATION_0`, so photos taken with the device
  turned are saved sideways. Fix: derive the rotation from device orientation.
- **`HW-18-0030`** (B/medium, confirmed, systematic): `requestPermissionsFromUser`'s
  result is ignored at `Index.ets:64-74`; the pipeline is built on denial too and
  the promise has no `.catch`. Fix: check `authResults` first.
- **`HW-18-0041`** (B/medium, probable, systematic): camera init is deferred by a
  fixed 200 ms `setTimeout` with no check that `surfaceId` is non-empty, and
  `onForeground` adds a third init path. Fix: one owner, `onLoad` after the
  grant.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-camera-cameramanager.md` - `getSupportedOutputCapability`, `createPreviewOutput`, `createPhotoOutput`, `createSession`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-camera-cameramanager
- `documentation/harmonyos-references/04_media/arkts-apis-camera-i.md` - `Profile`, `CameraOutputCapability`, `PhotoCaptureSetting`, `QualityLevel`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-camera-i
- `documentation/harmonyos-references/04_media/arkts-apis-camera-photooutput.md` - `capture`, `photoAssetAvailable`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-camera-photooutput
- `documentation/harmonyos-guides/02_media/camera-shooting.md` - the session configuration order
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-shooting
- `documentation/harmonyos-guides/05_media/camera-rotation-angle-adaptation.md` - the capture rotation this sample hardcodes
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-rotation-angle-adaptation
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - `READ/WRITE_IMAGEVIDEO`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `documentation/harmonyos-references/07_ai/core-vision-face-detector-api.md` - `faceDetector`, for the feature the *document* describes (no sample shipped)
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/core-vision-face-detector-api
- `documentation/harmonyos-references/06_standard-libraries/opengles.md` - OpenGL ES, likewise document-only here
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/opengles
- `documentation/harmonyos-guides/03_application-framework/arraybuffer-object.md` - the native round-trip the document sketches
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arraybuffer-object
- `PHOTO-26` - the doc this sample actually belongs to (自定义相机分辨率设置)
- `PHOTO-29` - the device-orientation mapping missing from `capture()`
- `PHOTO-30` - the picture-in-picture answer to `onBackground → releaseCamera`
