---
id: PHOTO-26
title: Preview resolution switch - pick a profile from the camera's own list and rebuild the session
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/26_insurance-v1_2-ts_32.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/insurance-v1_2-ts_32-0000002312678754
sample: huawei_industry_tree/18_photography/downloads/CameraResolution.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkGraphics2D", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CameraKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit", "@kit.PreviewKit"]
apis: [abilityAccessCtrl, camera, colorSpaceManager, common, curves, dataSharePredicates, display, filePreview, hilog, photoAccessHelper]
permissions: [ohos.permission.CAMERA, ohos.permission.WRITE_IMAGEVIDEO, ohos.permission.READ_IMAGEVIDEO]
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0008, HW-18-0010, HW-18-0030]
status: verified-with-fixes
---

## When to use

Load this card when a camera UI needs a **quality or resolution toggle** - the
HD/SD switch that appears in the top bar of most camera apps. The pattern:
never hardcode a resolution. Ask `getSupportedOutputCapability` for the
device's `previewProfiles`, filter them to the aspect ratio your layout is
built for, take the smallest for the low setting and the largest for the high
one, then rebuild the session with the chosen profile.

Two things make this more than a menu. First, a preview profile cannot be
swapped on a running session - `createPreviewOutput` takes the profile, so
changing it means tearing the session down and building a new one. Second, the
photo profile has to follow the preview profile's aspect ratio, or the captured
image will be framed differently from what the user saw.

It generalises to any capability-driven setting on the camera: frame rate,
colour space, stabilisation mode. The shape is always the same - query what the
device supports, present only that, and treat the session as disposable.

**This document is published under the slug `insurance-v1_2-ts_32`**
(`HW-18-0008`) - a wrong-domain URL from an unrelated pipeline. The content is
photography; only the address is wrong. Cite the title, not the slug.

## Feature checklist

- The preview fills a 4:3-ish surface sized from the display width and a fixed
  ratio constant.
- A top bar carries a resolution toggle plus four decorative camera-parameter
  icons.
- Touching the toggle switches between the lowest and highest preview profile
  at the target aspect ratio, and the icon changes to reflect the new state.
- The switch rebuilds the camera session; the preview resumes at the new
  resolution.
- The shutter captures at `QUALITY_LEVEL_HIGH` in high-resolution mode and
  `QUALITY_LEVEL_LOW` otherwise.
- Captured photos are written to the gallery and the thumbnail in the bottom
  left updates.
- Tapping the thumbnail opens the photo in the system Photos app; returning
  closes the preview overlay.
- Two-finger pinch zooms within the device's reported zoom range.

## Architecture

One `entry` module, two real source files: the page and the camera helper.

```
entry/src/main/ets
├── constants/CameraConstants.ets   ratios, icon sizes, and EPSILON = 0.03
├── entryability/EntryAbility.ets
├── entrybackupability/
├── pages/Index.ets                 @Entry, 221 lines: surface, top bar, shutter, thumbnail
└── utils/CameraShooter.ets         234 lines: profile selection, session build, capture, release
```

The documented tree matches the zip.

**The design decision worth copying** is that `cameraShooting` is idempotent and
takes the setting as an argument: `cameraShooting(highResolution, surfaceId, context)`
releases whatever session exists, picks the profile for the requested quality,
and builds a fresh one. The page never manipulates the session directly - it
calls the same function on first appearance and on every toggle, with a
different boolean. That is the right factoring for a capability toggle, because
"apply this setting" and "start the camera" are genuinely the same operation
once the session is disposable.

**The structure worth avoiding** is what surrounds it. `previewOutput`,
`cameraInput`, `photoSession`, `photoOutPut`, `currentContext`, `resolution`
and `uri` are module-level `let`s, and `releaseCamera()` is called on line 40
of `cameraShooting` **without `await`** even though it is `async`. So every
toggle starts building a new session while the old input is still closing
(`HW-18-0010`), and the truthiness guards inside `releaseCamera` keep firing on
already-released handles because nothing is ever set back to `undefined`.

## Implementation steps

1. **Declare CAMERA, READ_IMAGEVIDEO and WRITE_IMAGEVIDEO** with `reason` and
   `usedScene`; request exactly that set at runtime.
2. **Branch on the request result** before building the pipeline, and add a
   `.catch` - the sample does neither (`HW-18-0030`).
3. **Take the surface id in `XComponent.onLoad`,** after
   `setXComponentSurfaceRotation({ lock: true })` and
   `setXComponentSurfaceRect(...)`.
4. **Query `getSupportedOutputCapability(device, SceneMode.NORMAL_PHOTO)`** and
   read `previewProfiles`.
5. **Match the aspect ratio with a tolerance,** not with `===` - reported sizes
   do not divide to exactly 16/9 or 4/3.
6. **Scan forward for the low profile and backward for the high one**; the
   array is ordered, so first-match and last-match give the two ends of the
   range at that ratio.
7. **Pick the photo profile by the same ratio test** so the capture is framed
   like the preview.
8. **Await the teardown before the rebuild** and null the module fields
   (`HW-18-0010`).
9. **Map the setting onto `PhotoCaptureSetting.quality`** as well as the
   preview profile - resolution and encode quality are separate knobs.

## Verified snippets

All snippets are from `CameraResolution.zip`. Corrected forms are marked.

**Selecting the two profiles — `entry/src/main/ets/utils/CameraShooter.ets`** (as shipped)

```typescript
let photoSize: Size = { width: 1920, height: 1080 };   // the target aspect, not a target size

// 获取支持的相机输出能力
let cameraOutputCap: camera.CameraOutputCapability =
  cameraManager.getSupportedOutputCapability(cameraArray[0], camera.SceneMode.NORMAL_PHOTO);
// 获取预览流的Profile列表
let previewProfilesArray: camera.Profile[] = cameraOutputCap.previewProfiles;

// 获取低分辨率profile - first match, scanning forward
let lowResolutionProfile: camera.Profile | undefined = undefined;
let highResolutionProfile: camera.Profile | undefined = undefined;
for (let i = 0; i < previewProfilesArray.length; i++) {
  if (Math.abs(previewProfilesArray[i].size.width / previewProfilesArray[i].size.height -
    photoSize.width / photoSize.height) < CameraConstants.EPSILON) {
    lowResolutionProfile = previewProfilesArray[i];
    break;
  }
}
// 获取高分辨率profile - last match, scanning backward
for (let i = previewProfilesArray.length - 1; i >= 0; i--) {
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
// 选择合适的预览流分辨率
let previewProfile = highResolution ? highResolutionProfile : lowResolutionProfile;
previewOutput = cameraManager.createPreviewOutput(previewProfile, surfaceId);
```

**Three details carry this.** `photoSize` is named like a resolution but is only
ever used as a **ratio**: `1920 / 1080` is the 16:9 target, and no code ever
asks for those exact pixels. `EPSILON = 0.03` is what makes the comparison
work at all - a device reporting 1440x1080 or 2312x1080 divides to something
near but not equal to the nominal ratio, so an equality test would find
nothing. And the two loops differ only in direction: the profile array is
ordered by size, so first-match at the target ratio is the smallest and
last-match is the largest, which gives a low/high pair without sorting.

The `undefined` check afterwards is the piece most implementations skip. A
device that offers no profile at your layout's ratio is a real case, and the
right response is to fail visibly rather than pass `undefined` into
`createPreviewOutput`.

**Matching the photo profile to the preview — same file** (as shipped)

```typescript
// 获取支持的拍照流profile
let photoProfilesArray: camera.Profile[] = cameraOutputCap.photoProfiles.slice().reverse();
let photoProfile: camera.Profile | undefined = undefined;
if (previewProfile !== undefined) {
  // 选择与预览流分辨率宽高比一致的拍照流分辨率
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

**`.slice().reverse()` before `.find()` is "take the largest matching photo
profile".** `slice()` copies so the capability object is not mutated, `reverse()`
puts the biggest first, and `find` then returns the highest-resolution photo
profile at the preview's aspect ratio. The capture always runs at full
resolution regardless of the preview setting - which is correct: the preview
toggle is about what the surface renders, not about degrading the saved photo.

Note that the photo profile is matched against `photoSize`'s ratio rather than
against `previewProfile.size` - functionally the same here, since the preview
profile was itself selected by that ratio, but it means the two selections do
not actually depend on each other.

**Applying the setting and the quality — `entry/src/main/ets/pages/Index.ets` and the helper** (corrected, see `HW-18-0010`, `HW-18-0030`)

```typescript
// Index.ets - the toggle
Image(this.currentImage)
  .height(CameraConstants.OPERATOR_SIZE)
  .animation({ curve: curves.springMotion() })
  .onTouch((event: TouchEvent) => {
    if (event.type === TouchType.Down) {
      this.currentImage = highQuality ? this.clickImages[1] : this.clickImages[0];
      highQuality = !highQuality;
      cameraShooting(highQuality, surfaceId, this.context);   // same call as on first appearance
    }
    if (event.type === TouchType.Up) {
      this.currentImage = highQuality ? this.unclickImages[1] : this.unclickImages[0];
    }
  })

// Index.ets - aboutToAppear
async aboutToAppear() {
  try {
    const data = await abilityAccessCtrl.createAtManager()
      .requestPermissionsFromUser(this.context, this.permissions);
    if (data.authResults.some((r: number) => r !== 0)) {   // FIX: sample ignores the result entirely
      hilog.error(DOMAIN, TAG, 'Camera permission denied');
      return;
    }
    setTimeout(async () => {
      zoomRatioRange = await cameraShooting(highQuality, surfaceId, this.context);
    }, 200);
  } catch (error) {
    hilog.error(DOMAIN, TAG, `The request call failed. error: ${JSON.stringify(error)}`);
  }
}

// CameraShooter.ets - the top of cameraShooting
export async function cameraShooting(highResolution: boolean, surfaceId: string, context: Context): Promise<number[]> {
  let zoomRatioRange: number[] = [];
  currentContext = context;
  resolution = highResolution;
  await releaseCamera();          // FIX: shipped call is not awaited - it races the rebuild below
  // ... build the session with the chosen profile ...
}

// CameraShooter.ets - capture
export async function capture() {
  let settings: camera.PhotoCaptureSetting = {
    quality: resolution ? camera.QualityLevel.QUALITY_LEVEL_HIGH : camera.QualityLevel.QUALITY_LEVEL_LOW,
    rotation: camera.ImageRotation.ROTATION_0,
    mirror: false
  };
  photoOutPut.capture(settings);
}
```

**One boolean drives three things**: which preview profile is used, which icon
the toggle shows, and the JPEG quality level on capture. Keeping it in a
module-level `resolution` that `cameraShooting` writes on every call means
`capture()` cannot disagree with the live session - a small but real benefit of
routing the setting through the rebuild function rather than reading page state
at capture time.

The correction on `releaseCamera` is the load-bearing one. It is `async` and its
body is a chain of `stop`/`close`/`release` calls, none awaited; the caller does
not await it either, so `createCameraInput` and `commitConfig` for the new
session run while the previous input is still closing. On a real device that
surfaces as an intermittent camera-occupied error or a black preview after a
fast double toggle. The same defect ships in nine photography samples
(`HW-18-0010`); the fix is to await each release, set the fields back to
`undefined`, and await the helper at its call sites.

The `setTimeout(..., 200)` before the first `cameraShooting` is a wait for the
`XComponent` to publish its surface id. It works but is a race; keying off
`onLoad` instead, as `PHOTO-27` does, is deterministic.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.CAMERA",           "reason": "$string:reason_camera",
    "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.WRITE_IMAGEVIDEO", "reason": "$string:reason_write_imagevideo",
    "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.READ_IMAGEVIDEO",  "reason": "$string:reason_read_imagevideo",
    "usedScene": { "abilities": ["EntryAbility"] } }
]
```

- The runtime array in `Index.ets` lists the same three, in a different order -
  which is fine; what matters is that it is a subset of the declared set.
  Compare `PHOTO-25`, where one extra undeclared entry fails the whole grant.
- READ_IMAGEVIDEO and WRITE_IMAGEVIDEO are **restricted** permissions and need
  an ACL entry in the signing profile.
- No `when` is given, so `inuse` applies - correct for a foreground camera.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- Only the rear camera is used: `cameraArray[0]` throughout, and the
  camera-switch icon's `onClick` is an empty body with a comment inviting the
  developer to implement it.
- The surface rect is computed once from `display.getDefaultDisplaySync().width`
  and `SURFACE_RATIO = 632 / 400`, so it does not follow a window resize or a
  fold.
- `setColorSpace(DISPLAY_P3)` is applied unconditionally, with no check against
  the session's supported colour spaces.
- The flash, torch and white-balance icons in the top bar are decorative; the
  document says so directly (其他相机参数元素仅为还原布局效果 - the other camera
  parameter elements only reproduce the layout).
- `getThumbnail()` exists on the page but nothing calls it; the thumbnail is
  filled by `AppStorage.setOrCreate('photoUri', ...)` from the
  `photoAssetAvailable` callback, which is the better path anyway - it uses the
  captured asset rather than the newest asset in the gallery. Contrast
  `PHOTO-25`'s `HW-18-0074`.

## Pitfalls

- **`HW-18-0008`** (E/low, confirmed): this page is published at
  `architecture-guides/insurance-v1_2-ts_32-…` - an insurance slug on a camera
  document. The OpenGL face-box page (`PHOTO-31`) is likewise under an `audio`
  slug, and four more instances exist across `13_media_entertainment` and
  `15_utilities`. The pages were cloned from other industries' shells and never
  re-keyed, so the document ids are misleading and slug-based navigation
  breaks. Fix: re-publish under topic-correct slugs with redirects.
- **`HW-18-0010`** (B/medium, confirmed): `releaseCamera` awaits none of its
  `stop`/`close`/`release` calls and nulls no fields, and `cameraShooting`
  calls it without `await` immediately before rebuilding - so every resolution
  toggle races the teardown, and the guards inside `releaseCamera` stay truthy
  on released objects. Nine samples in this industry. Fix: await each release,
  reset the fields to `undefined`, await the call.
- **`HW-18-0030`** (B/medium, confirmed): `requestPermissionsFromUser` resolves
  into a `.then(() => ...)` that takes no parameter and has no `.catch`, so
  denying the dialog still builds the camera pipeline and produces raw
  permission errors instead of a refused-state UI. Six samples. Fix: check
  `data.authResults`.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-camera-cameramanager.md` - `getSupportedOutputCapability`, `createPreviewOutput`, `createPhotoOutput`, `createSession`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-camera-cameramanager
- `documentation/harmonyos-references/04_media/arkts-apis-camera-photosession.md` - `beginConfig`, `commitConfig`, `setColorSpace`, `getZoomRatioRange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-camera-photosession
- `documentation/harmonyos-references/04_media/arkts-apis-camera-previewoutput.md` - the preview output and its profile
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-camera-previewoutput
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-xcomponent.md` - `setXComponentSurfaceRect` and `getXComponentSurfaceId`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- `documentation/harmonyos-guides/05_media/camera-shooting-case.md` - the reference photo-session pipeline
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-shooting-case
- `documentation/harmonyos-guides/05_media/camera-preconfig.md` - the preconfigured alternative to hand-picking profiles
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-preconfig
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` - CAMERA
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - READ_IMAGEVIDEO, WRITE_IMAGEVIDEO
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `PHOTO-25` - preview profile selection by proximity to the window size, not by array ends
- `PHOTO-27` - the same session build driven by page lifecycle instead of a setting
