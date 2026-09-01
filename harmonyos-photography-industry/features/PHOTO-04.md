---
id: PHOTO-04
title: Photo aspect-ratio switch - rebuild the camera session with matched preview and photo profiles
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/04_ratio_camera.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/ratio_camera-0000002252528422
sample: huawei_industry_tree/18_photography/downloads/RatioCamera.zip
kits: ["@kit.CameraKit", "@kit.MediaLibraryKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.ArkGraphics2D", "@kit.ArkData", "@kit.SensorServiceKit", "@kit.PreviewKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["camera.getCameraManager", getSupportedCameras, createCameraInput, getSupportedOutputCapability, createPreviewOutput, createPhotoOutput, createSession, beginConfig, commitConfig, setColorSpace, setFlashMode, setFocusMode, getZoomRatioRange, PhotoCaptureSetting, "photoOutput.capture", photoAssetAvailable, MediaAssetChangeRequest, saveCameraPhoto, XComponent, XComponentController, setXComponentSurfaceRect, "sensor.on", GravityResponse, "abilityAccessCtrl.requestPermissionsFromUser", requestPermissionOnSetting, "filePreview.closePreview"]
permissions: ["ohos.permission.CAMERA", "ohos.permission.WRITE_IMAGEVIDEO", "ohos.permission.READ_IMAGEVIDEO"]
min_api: 20
modules: [entry]
findings: [HW-18-0004, HW-18-0010, HW-18-0026, HW-18-0027]
status: verified-with-fixes
---

## When to use

Load this card when building a **custom camera whose viewfinder changes shape**
- the 1:1 / 16:9 (or 4:3, 3:2) selector every camera app has along the top of
the frame. It is also the smallest complete Camera Kit pipeline in this
industry pack: permission, input, two outputs, session, capture, save.

The thing to understand before writing any of it: **an aspect ratio is not a
crop, it is a different session.** The preview `Profile` and the photo
`Profile` both carry a fixed `size`, they are chosen from what the device
reports in `getSupportedOutputCapability`, and neither can be changed on a live
session. So switching ratio means tearing the whole pipeline down and building
it again with different profiles - and resizing the `XComponent` surface to
match, or the preview is stretched.

That generalises to every other capability toggle on a camera: resolution,
HDR, video vs photo mode, front vs back. They are all "release, re-select
profiles, rebuild", and the only difference is which profile field you filter
on. If you get the teardown ordering right once, all of them work.

## Feature checklist

- Full-screen live preview on an `XComponent` surface, black background.
- Two ratio buttons (1:1 and 16:9); the selected one shows in blue, the other
  in white.
- Switching ratio rebuilds the camera session and resizes the preview surface
  to `width x width` or `width x width*16/9`, adjusting the top margin so the
  frame stays visually centred.
- A flash menu (off / on) bound to the flash icon, hidden on the front camera.
- A front/back flip button that rebuilds the session and enables mirroring for
  self-portraits.
- A shutter button; captured photos land in the system gallery directly from
  the camera pipeline.
- A round thumbnail of the last shot, which rotates with the device (gravity
  sensor) and opens the system Photos app when tapped.
- If CAMERA is denied, every control toasts 无相机权限 and a button offers to
  open the permission settings sheet.
- Leaving the page releases the camera; returning to the foreground rebuilds it.

## Architecture

One `entry` module, four source files, 621 lines total.

```
entry/src/main/ets
├── constants/DegreeConstants.ets   the four 45/135/225/315 sensor quadrant bounds
├── entryability/EntryAbility.ets   full-screen, avoid areas -> AppStorage, onForeground -> fromBack()
├── pages/Index.ets                 @Entry: the whole UI, permission flow, sensor, ratio state
└── utils/CameraShooter.ets         the camera pipeline: 4 module-level handles + 6 functions
```

The documented tree matches the zip apart from one name: it lists
`entrybackupability/EntryBackAbility.ets`, and the zip has no
`entrybackupability` directory at all.

**The design decision worth copying** is that `CameraShooter.ets` holds the
four camera handles as **module-level `let`s**, not as fields on the page:

```typescript
let previewOutput: camera.PreviewOutput;
let cameraInput: camera.CameraInput;
let photoSession: camera.PhotoSession;
let photoOutPut: camera.PhotoOutput;
```

The camera is a single hardware resource with one owner, and a module is a
better model of that than a component that can be constructed twice. It also
lets `EntryAbility.onForeground` call the exported `fromBack(context)` to
rebuild the pipeline without holding a reference to the page. The page keeps
only UI state - which ratio, which camera, which flash icon - and every camera
operation is a free function call.

**The decision worth avoiding** in the same file: the handles are declared but
never reset. `releaseCamera` releases each object and leaves the variable
pointing at it, so `if (photoSession)` is still true afterwards and a second
release runs against a dead object. Combined with the un-awaited teardown
(`HW-18-0010`) this is the sample's most consequential defect - see the
corrected form below.

## Implementation steps

1. **Request CAMERA in `onPageShow`, await it, and only then start the
   pipeline.** Accumulate the check across every permission - do not `return`
   from inside the first loop iteration (`HW-18-0026`).
2. **Give the surface its size before asking for its id.** `onAttach` calls
   `setXComponentSurfaceRect` and then `getXComponentSurfaceId`; the surface
   dimensions are what the preview profile has to agree with.
3. **Delay the first `cameraShooting` by a tick** (`setTimeout(..., 200)` in
   the sample) so the `XComponent` has published a real surface id.
4. **Await the teardown before rebuilding**, and null the handles inside it
   (`HW-18-0010`).
5. **Select the preview profile from the device's `previewProfiles`**, filtered
   by the wanted ratio, and **guard the result before creating the output** -
   `find` returns `undefined` on a device with a different profile set.
6. **Select the photo profile with the same ratio** so preview and capture
   frame the same scene.
7. **Configure inside `beginConfig` / `commitConfig`**: add the input, both
   outputs, and the colour space; then `start()`.
8. **Register `photoAssetAvailable` before committing** - the callback saves
   straight into the gallery via `MediaAssetChangeRequest.saveCameraPhoto()`,
   which needs no album permission.
9. **Feed the sensor-derived rotation into `PhotoCaptureSetting`** instead of
   the hardcoded `ROTATION_0` (`HW-18-0027`).
10. **Resize the surface again on every ratio change** - the session rebuild
    alone does not move the `XComponent`.

## Verified snippets

All snippets are from `RatioCamera.zip`. Corrected forms are marked.

**Ratio-aware profile selection — `entry/src/main/ets/utils/CameraShooter.ets`** (corrected, guards added)

```typescript
let previewProfilesArray: camera.Profile[] = cameraOutputCap.previewProfiles;
let photoProfilesArray: camera.Profile[] = cameraOutputCap.photoProfiles;
let previewProfile: undefined | camera.Profile = previewProfilesArray.find((profile: camera.Profile) => {
  let screen = display.getDefaultDisplaySync();
  if (screen.width <= 1080) {
    if (ratio) {
      return profile.size.height === 1080 && profile.size.width === 1920;   // 16:9
    } else {
      return profile.size.height === 1080 && profile.size.width === 1080;   // 1:1
    }
  } else {
    if (ratio) {
      return profile.size.height === 1440 && profile.size.width === 2560;
    } else {
      return profile.size.height === 1440 && profile.size.width === 1440;
    }
  }
});
if (previewProfile === undefined) {
  previewProfile = previewProfilesArray[0];        // FIX: sample passes undefined into create
}
let photoProfile: undefined | camera.Profile = photoProfilesArray.find((profile: camera.Profile) => {
  if (ratio) {
    return profile.size.height === 1080 && profile.size.width === 1920;
  }
  return profile.size.height === 1080 && profile.size.width === 1080;
});
if (photoProfile === undefined) {
  photoProfile = photoProfilesArray[0];            // FIX: same
}
previewOutput = cameraManager.createPreviewOutput(previewProfile, surfaceId);
photoOutPut = cameraManager.createPhotoOutput(photoProfile);
```

**`profile.size` is in sensor orientation, so `width` is the long edge.** A
16:9 portrait viewfinder is `1920 x 1080` here, and 1:1 is `1080 x 1080` -
the same height with the long edge cut back to square. That is the whole ratio
mechanism: keep the short edge, vary the long one.

The device split on `screen.width <= 1080` is the sample picking two known
profile pairs for two known phone classes. It is exactly the pattern
`HW-18-0021` flags across four sibling samples: `find` on hardcoded dimensions
returns `undefined` on any device that does not expose that precise profile,
and the shipped code passes it straight into `createPreviewOutput`, which
throws `7400101` inside an async function nobody catches - a black viewfinder
with no error on screen. The two guards above are the minimum fix; a real
implementation should score the candidates by ratio distance rather than
matching exact pixel counts.

The shipped `photoProfile` callback is worse than it looks: its body is
`if (previewProfile) { ... } return undefined;`, so it consults an outer
variable to decide whether to evaluate the element at all. When `previewProfile`
is `undefined` the predicate returns `undefined` for every element and the
`find` yields `undefined` too. The corrected form drops the outer test - the
guard above already handles that case.

**Session assembly and teardown — same file** (corrected, see `HW-18-0010`)

```typescript
photoSession = cameraManager.createSession(camera.SceneMode.NORMAL_PHOTO) as camera.PhotoSession;
photoSession.beginConfig();
photoSession.addInput(cameraInput);
photoSession.addOutput(previewOutput);
photoSession.addOutput(photoOutPut);
photoSession.setColorSpace(colorSpaceManager.ColorSpace.DISPLAY_P3);
await photoSession.commitConfig();
await photoSession.start();

export async function releaseCamera(): Promise<void> {
  if (photoSession) {
    await photoSession.stop();          // FIX: sample drops all five promises
  }
  if (cameraInput) {
    await cameraInput.close();
  }
  if (previewOutput) {
    await previewOutput.release();
  }
  if (photoSession) {
    await photoSession.release();
  }
  if (photoOutPut) {
    await photoOutPut.release();
  }
  photoSession = undefined;             // FIX: sample leaves the handles pointing at dead objects
  cameraInput = undefined;
  previewOutput = undefined;
  photoOutPut = undefined;
}
```

**The release order is not arbitrary.** Stop the session first so no frames are
in flight, close the input to hand the sensor back, then release the outputs
and the session. Doing it the other way round releases buffers a running
session still owns.

`RatioCamera` is one of the better samples here in that `cameraShooting` does
`await releaseCamera()` on entry - most of its siblings call it bare. But
`releaseCamera` itself awaits nothing, so the `await` returns as soon as the
five calls have been *issued*, and `createCameraInput`/`open()` on the next
line races a camera that is still closing. On a fast ratio-toggle or
front/back flip this surfaces as a camera-occupied error and a black preview.
The finding covers nine samples with this shape (`HW-18-0010`); this is the
version to copy after applying the fix.

`setColorSpace(DISPLAY_P3)` must sit between `beginConfig` and `commitConfig`
like everything else that configures the session.

**Capture with the real device rotation — same file** (corrected, see `HW-18-0027`)

```typescript
export function capture(isFront: boolean, rotation: camera.ImageRotation): void {
  let settings: camera.PhotoCaptureSetting = {
    quality: camera.QualityLevel.QUALITY_LEVEL_HIGH,
    rotation: rotation,                 // FIX: sample hardcodes camera.ImageRotation.ROTATION_0
    mirror: isFront
  };
  photoOutPut.capture(settings);
}

function setPhotoOutputCb(photoOutput: camera.PhotoOutput): void {
  photoOutput.on('photoAssetAvailable',
    async (_err: BusinessError, photoAsset: photoAccessHelper.PhotoAsset): Promise<void> => {
      let accessHelper: photoAccessHelper.PhotoAccessHelper =
        photoAccessHelper.getPhotoAccessHelper(currentContext);
      let assetChangeRequest: photoAccessHelper.MediaAssetChangeRequest =
        new photoAccessHelper.MediaAssetChangeRequest(photoAsset);
      assetChangeRequest.saveCameraPhoto();
      await accessHelper.applyChanges(assetChangeRequest);
      uri = photoAsset.uri;
      AppStorage.setOrCreate('photoUri', await photoAsset.getThumbnail());
    });
}
```

**`photoAssetAvailable` is the permission-free save path** and the reason this
sample needs no album permission at all. The camera framework hands the app a
`PhotoAsset` it already created; `saveCameraPhoto()` on a
`MediaAssetChangeRequest` commits it to the gallery. Nothing is decoded,
packed or written by the app, and `READ_IMAGEVIDEO` / `WRITE_IMAGEVIDEO` never
come into it - which is what makes their presence in `module.json5`
(`HW-18-0004`) both useless and disqualifying at app review.

The rotation is the sample's sharpest irony. `Index.ets` subscribes to the
gravity sensor and maps the device angle onto `camera.ImageRotation` through
four 90-degree quadrants - and then uses the result only to spin the thumbnail
image, while `capture()` writes `ROTATION_0` into every shot. Photos taken with
the phone on its side are saved upright-tagged and show sideways in the
gallery. The computed value is right there; pass it through.

**Ratio switch in the page — `entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
Image($r(this.imageSourceOne))                 // the 1:1 button
  .onClick(() => {
    if (this.imageSourceOne === 'app.media.onewhite' && this.isPermitted) {
      this.imageSourceOne = 'app.media.oneblue';
      this.imageSourceNine = 'app.media.ninewhite';
      this.cameraMargin = 100;                 // square frame sits lower
      this.cameraHeight = 1440;                // px, converted with px2vp on the XComponent
      this.isRatio = false;
      cameraShooting(cameraPosition, surfaceId, this.context, this.isRatio);
      this.Initialize();
    } else {
      this.getUIContext().getPromptAction().showToast({ message: '无相机权限' });
    }
  });

Initialize(): void {
  this.isStabilization = false;
  this.flashPic = $r('app.media.ic_camera_public_flash_off');
  this.isMovingPhoto = false;
  if (this.isRatio === true) {
    this.mXComponentController.setXComponentSurfaceRect({
      surfaceWidth: display.getDefaultDisplaySync().width,
      surfaceHeight: display.getDefaultDisplaySync().width * 16 / 9
    });
  } else {
    this.mXComponentController.setXComponentSurfaceRect({
      surfaceWidth: display.getDefaultDisplaySync().width,
      surfaceHeight: display.getDefaultDisplaySync().width
    });
  }
}
```

**`Initialize()` is the piece that is easy to forget.** The session rebuild
changes what the camera produces; `setXComponentSurfaceRect` changes what the
surface accepts. Skip it and the new 1:1 stream is stretched across the old
16:9 surface. It also resets the per-session UI toggles - flash, stabilisation,
moving-photo - because those live on the session that was just destroyed and
would otherwise show stale state.

Note the un-awaited `cameraShooting` call: the rebuild is fire-and-forget while
`Initialize()` resizes the surface underneath it. The two happen to interleave
acceptably, but the honest form is `await cameraShooting(...)` in an `async`
handler followed by `this.Initialize()`.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.CAMERA",         "reason": "$string:reason_camera",
    "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.WRITE_IMAGEVIDEO", "reason": "$string:reason_camera",
    "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.READ_IMAGEVIDEO",  "reason": "$string:reason_camera",
    "usedScene": { "abilities": ["EntryAbility"] } }
]
```

- **`CAMERA` is the only one this sample needs** and the only one it requests
  at runtime. The two album permissions are restricted (ACL) and are never
  requested - `HW-18-0004` catalogues nine samples sharing this template
  mistake. Delete both entries.
- `usedScene` has no `when` field. For a foreground-only camera, `"when":
  "inuse"` should be stated explicitly.
- All three share one `reason` string, so the system dialog would show a camera
  justification for an album permission.
- `deviceTypes` is `["phone"]` only, which is at least consistent with the
  hardcoded portrait geometry.
- The permanent-refusal fallback is present and correct: the visible-when-denied
  button calls `requestPermissionOnSetting`, which opens the settings sheet
  when the ordinary request would no longer show a dialog. It rebuilds the
  camera in the `then` when the status comes back `PERMISSION_GRANTED`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- The preview geometry is hardcoded in pixels (`1440` wide, `2560`/`1440` tall,
  margins of 40 and 100 vp) and the profile match is on exact pixel sizes. The
  sample is calibrated for one phone class and will pick the wrong profile - or
  none - elsewhere.
- `setColorSpace(DISPLAY_P3)` is applied unconditionally without checking
  `getSupportedColorSpaces`.
- The 200 ms `setTimeout` before the first `cameraShooting` is a race the
  sample papers over rather than solves; the correct trigger is the
  `XComponent`'s own `onAttach`/`onLoad`, which is already where the surface id
  is fetched.
- `getThumbnail(context)` is defined on the page and never called - the
  thumbnail comes from the capture callback instead. Dead code.
- No zoom, no focus tap, no video mode. For pinch zoom, the flash modes and
  live photo built on the same `CameraShooter` module, see `PHOTO-01`.

## Pitfalls

- **`HW-18-0026`** (B/low, confirmed): `requestPermission` returns from inside
  the first loop iteration - `if (grantStatus[i] !== 0) return false; else
  return true;` - so only `authResults[0]` is ever evaluated. With CAMERA
  granted and a later permission denied the helper still reports success. Fix:
  `return false` inside the loop on any denial, `return true` after it.
- **`HW-18-0027`** (B/low, confirmed): `capture()` hardcodes
  `rotation: camera.ImageRotation.ROTATION_0` although the page computes the
  real orientation from the gravity sensor and uses it only for the thumbnail.
  Photos taken with the device rotated are saved sideways. Systematic across
  three samples. Fix: pass the computed rotation into `PhotoCaptureSetting`.
- **`HW-18-0010`** (B/medium, confirmed): `releaseCamera` issues
  `stop`/`close`/`release` without awaiting any of them and never nulls the
  handles, so the rebuild races the teardown and the truthiness guards stay
  true on released objects. Systematic across nine photography samples. Fix:
  await each call in order and reset the handles to `undefined`.
- **`HW-18-0004`** (D/medium, confirmed): `READ_IMAGEVIDEO` and
  `WRITE_IMAGEVIDEO` are declared although the sample saves exclusively through
  the permission-free `photoAssetAvailable` / `saveCameraPhoto` path and never
  requests them. Restricted permissions block app review. Fix: delete both
  entries from `module.json5`.
- **Unguarded hardcoded profiles** (same shape as `HW-18-0021`, which names
  four sibling samples): `find` on exact pixel dimensions can return
  `undefined`, and the shipped guards test the *output* after
  `createPreviewOutput`/`createPhotoOutput` have already been called with it.
  Fix: guard the profiles before the create calls and fall back to a supported
  one.
- **`photoProfile`'s predicate returns `undefined` for every element** when
  `previewProfile` is `undefined`, because it branches on an outer variable
  instead of the element it was given. Fix: drop the outer test.
- **The ratio buttons key off the icon resource string** (`if
  (this.imageSourceOne === 'app.media.onewhite')`) rather than off `isRatio`.
  The UI's own presentation is the state. Fix: branch on `this.isRatio`.
- **`fromBack()` on `onForeground` always rebuilds at 16:9** (`cameraShooting(..., true)`),
  so backgrounding the app in 1:1 mode returns to a 16:9 stream while the
  buttons still show 1:1 selected.

## References

- `documentation/harmonyos-references/04_media/js-apis-camera.md` - `Profile`, `CameraOutputCapability`, `PhotoCaptureSetting`, `ImageRotation`, session lifecycle
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-camera
- `documentation/harmonyos-guides/02_media/camera-kit.md` - the Camera Kit model and the permission requirement
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-kit
- `documentation/harmonyos-guides/02_media/camera-preview.md` - binding a preview output to an `XComponent` surface id
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-preview
- `documentation/harmonyos-guides/02_media/camera-shooting.md` - `photoAssetAvailable` and the capture flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-shooting
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `MediaAssetChangeRequest.saveCameraPhoto`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - `XComponentController.setXComponentSurfaceRect`, `getXComponentSurfaceId`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` - `requestPermissionsFromUser`, `authResults`, `requestPermissionOnSetting`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - `READ_IMAGEVIDEO` / `WRITE_IMAGEVIDEO` and why they do not belong here
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `documentation/harmonyos-references/03_system/sensor-service-arkts.md` - `sensor.on(SensorId.GRAVITY)` and `GravityResponse`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/sensor-service-arkts
- `PHOTO-01` - the same `CameraShooter` module with pinch zoom, live photo and flash modes on top
