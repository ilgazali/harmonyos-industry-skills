---
id: OFFICE-07
title: ID photo capture with a mask overlay - XComponent preview under a Stack cut-out frame
industry: 05_office
doc: huawei_industry_tree/05_office/docs/07_camera_page.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/camera_page-0000002245028040
sample: huawei_industry_tree/05_office/downloads/MaskCamera.zip
kits: ["@kit.CameraKit", "@kit.MediaLibraryKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [XComponent, XComponentType, XComponentController, "XComponentController.setXComponentSurfaceRect", "XComponentController.getXComponentSurfaceId", ImageAIOptions, ImageAnalyzerController, ImageAnalyzerType, Stack, "camera.getCameraManager", "CameraManager.getSupportedCameras", "CameraManager.getSupportedSceneModes", "CameraManager.getSupportedOutputCapability", "CameraManager.createCameraInput", "CameraManager.createPreviewOutput", "CameraManager.createPhotoOutput", "CameraManager.createSession", "camera.SceneMode.NORMAL_PHOTO", "CameraInput.open", "CameraInput.close", "CameraInput.on('error')", "PhotoSession.beginConfig", "PhotoSession.addInput", "PhotoSession.addOutput", "PhotoSession.commitConfig", "PhotoSession.start", "PhotoSession.stop", "PhotoSession.release", "PhotoSession.hasFlash", "PhotoSession.isFlashModeSupported", "PhotoSession.setFlashMode", "PhotoSession.isFocusModeSupported", "PhotoSession.setFocusMode", "PhotoSession.getZoomRatioRange", "PhotoSession.setZoomRatio", "PhotoSession.setSmoothZoom", "PhotoOutput.capture", "PhotoOutput.enableMovingPhoto", "PhotoOutput.on('photoAssetAvailable')", "camera.PhotoCaptureSetting", "photoAccessHelper.getPhotoAccessHelper", "photoAccessHelper.MediaAssetChangeRequest", "MediaAssetChangeRequest.saveCameraPhoto", "PhotoAccessHelper.applyChanges", "PhotoAsset.getThumbnail", "abilityAccessCtrl.createAtManager", "AtManager.checkAccessToken", "AtManager.requestPermissionsFromUser", "AtManager.requestPermissionOnSetting", "bundleManager.getBundleInfoForSelf", Navigation, NavPathStack, NavDestination]
permissions: ["ohos.permission.CAMERA"]
min_api: 20
modules: [entry]
findings: [HW-05-0037, HW-05-0038, HW-05-0039, HW-05-0040, HW-05-0041, HW-05-0042, HW-05-0043, HW-05-0044]
status: verified-with-fixes
---

## When to use

Load this card when an office app has to capture a **document photo inside a
fixed frame** - an ID card for identity verification, and by swapping the mask
asset a bank card, insurance card or badge.

The distinguishing requirement is the **mask**: the user must see a live camera
preview with a card-shaped cut-out, dimmed surroundings, rotated guidance text
and a warning strip, so that the captured photo is already correctly framed.
That rules out the system camera picker and forces a **custom capture pipeline**:
an `XComponent` surface driven by Camera Kit, with the mask laid over it in a
`Stack`.

The whole page is landscape-oriented by rotating the overlay elements 90
degrees rather than by rotating the window.

## Feature checklist

The application must be able to:

- Collect the name and ID number, and offer a front-side and a reverse-side
  capture entry.
- Request the camera permission with a two-stage flow (dialog, then Settings) and
  refuse to shoot when it is not granted.
- Render a live preview on an `XComponent` surface at a 4:3 ratio.
- Overlay a full-preview-sized mask image with the card cut-out, plus rotated
  instruction text and a warning strip.
- Switch the mask between front and reverse side based on the navigation
  parameter.
- Pick a preview profile close to 4:3 and no wider than 1920, and a photo profile
  matching the chosen preview size.
- Configure auto flash and continuous auto focus when the device supports them.
- Capture at high quality, save the asset into the media library and show its
  thumbnail.
- Open the captured photo in the system Gallery.
- Release the whole camera pipeline when the page is hidden, and rebuild it when
  it is shown again.

## Architecture

Single `entry` HAP:

| File | Responsibility |
| --- | --- |
| `pages/Index.ets` | `@Entry`; the ID-information form, the front/reverse capture entries, owns the `NavPathStack` (`@Provide pathStack`) and the `pageMap` builder |
| `pages/CameraPage.ets` | the capture `NavDestination`: `XComponent` + mask `Stack`, shutter, thumbnail, `onShown` / `onHidden` lifecycle |
| `utils/CameraShooter.ets` | the entire Camera Kit pipeline as free functions over module-level state |
| `utils/PermissionsView.ets` | the permission ladder (check -> dialog -> Settings), exported as a singleton |
| `constants/CameraConstants.ets` | preview/photo sizes and layout constants |
| `entryability/EntryAbility.ets`, `entrybackupability/EntryBackupAbility.ets` | ability entry and backup stub |

`CameraShooter.ets` deliberately holds the pipeline in **module-level
variables** (`previewOutput`, `cameraInput`, `photoSession`, `photoOutPut`,
`currentContext`, `uri`), so there is exactly one camera pipeline per process and
every exported function operates on it.

The mask is a `Stack` with two children of identical size:

```
Stack (height 558)
  XComponent(SURFACE)           width/height = previewSize px, 4:3
  Stack                          width/height = previewSize px
    Image(front|reverse frame)   the cut-out artwork, ImageFit.Fill
    Text/Span guidance           rotated 90 deg, translated -150 x
    Row(warning icon + text)     rotated 90 deg, translated +150 x
```

The comment in the sample states the constraint plainly: the mask asset must
cover the whole camera picture, because otherwise the semi-transparent ring
around the card outline cannot be produced.

Capture-to-gallery flow:

```
shutter onClick
  -> PermissionsView.checkPermissions(['ohos.permission.CAMERA'])
  -> capture() -> photoOutPut.capture({ quality: HIGH, rotation: ROTATION_0 })
  -> photoOutput.on('photoAssetAvailable') fires
       -> photoAccessHelper.getPhotoAccessHelper(currentContext)
       -> new MediaAssetChangeRequest(photoAsset).saveCameraPhoto()
       -> accessHelper.applyChanges(request)
       -> uri = photoAsset.uri
       -> AppStorage.setOrCreate('photoUri', await photoAsset.getThumbnail())
  thumbnail onClick -> previewPhoto() -> startAbility(com.huawei.hmos.photos)
```

## Implementation steps

1. **Declare only the permissions the scenario uses**, each with a complete
   `usedScene` including `when` (HW-05-0041), and keep the document's permission
   list in step with `module.json5` (HW-05-0040).
2. **Build the permission ladder.** `checkAccessToken` via
   `bundleManager.getBundleInfoForSelf(...).appInfo.accessTokenId`, then
   `requestPermissionsFromUser`, then `requestPermissionOnSetting` when
   `dialogShownResults[0]` is not `true`. Cast the context on **both** calls
   (HW-05-0042).
3. **Lay out the preview.** An `XComponent` of type `SURFACE` with an
   `XComponentController`; in `onLoad` call `setXComponentSurfaceRect` with the
   preview width/height, then `getXComponentSurfaceId()`.
4. **Start the camera once the surface exists.** Drive initialisation from the
   surface-ready path only, and guard on a non-empty surface id rather than a
   fixed delay (HW-05-0043).
5. **Release before rebuilding.** `cameraShooting` begins with a teardown; that
   teardown must be awaited, and every release inside it must be awaited too
   (HW-05-0038).
6. **Select the profiles.** Walk `cameraOutputCap.previewProfiles`, keep the
   widest profile whose `height/width` is near 0.75 and whose width is at most
   1920, then find the `photoProfile` whose size equals the chosen preview size.
7. **Assemble the session.** `createCameraInput` -> `on('error')` ->
   `open()` -> `createPreviewOutput(profile, surfaceId)` ->
   `createPhotoOutput(photoProfile)` -> `createSession<camera.PhotoSession>(NORMAL_PHOTO)`
   -> `beginConfig` -> `addInput` -> `addOutput` (preview) -> `addOutput` (photo)
   -> `commitConfig` -> `start`.
8. **Register the asset callback before the session starts.** `photoOutput.on('photoAssetAvailable', ...)`
   is what turns a shutter press into a gallery item; pair every `on(...)` in this
   file with an `off(...)` in the teardown (HW-05-0039).
9. **Configure capabilities defensively.** Check `hasFlash()` before
   `isFlashModeSupported(FLASH_MODE_AUTO)` before `setFlashMode`; check
   `isFocusModeSupported(FOCUS_MODE_CONTINUOUS_AUTO)` before `setFocusMode`; check
   `getZoomRatioRange().length` before `setZoomRatio`.
10. **Build the mask on top.** A second `Stack` sized to the same pixel
    dimensions as the `XComponent`, holding the frame `Image` and the rotated
    guidance elements. Use the sample's resource names, not the document's
    snippet (HW-05-0044).
11. **Save through the media library.** In the `photoAssetAvailable` callback,
    `new MediaAssetChangeRequest(photoAsset).saveCameraPhoto()` then
    `applyChanges`, keep `photoAsset.uri` for the Gallery jump, and publish
    `await photoAsset.getThumbnail()` into `AppStorage` for the corner preview.
12. **Log with a valid domain.** Use a domain inside `[0x0, 0xFFFF]` and the
    documented `(domain, tag, format, ...args)` order everywhere - the shipped
    file uses `-1` throughout, which prints nothing (HW-05-0037).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### XComponent preview with the mask Stack

`MaskCamera.zip#entry/src/main/ets/pages/CameraPage.ets`

```ts
let previewSize: camera.Size = {
  width: CameraConstants.PREVIEW_WIDTH,   // 1228.8
  height: CameraConstants.PREVIEW_HEIGHT  // 1638.4
};
let cameraPosition = 0; // rear camera by default
let surfaceId = '';

@Entry
@Component
export struct CameraPage {
  mXComponentController: XComponentController = new XComponentController;
  private aiController: ImageAnalyzerController = new ImageAnalyzerController();
  private options: ImageAIOptions = {
    types: [ImageAnalyzerType.SUBJECT, ImageAnalyzerType.TEXT],
    aiController: this.aiController
  };
  @State isIDCardFront: number = 0;

  // ...
  Stack() {
    XComponent({
      type: XComponentType.SURFACE,
      controller: this.mXComponentController,
      imageAIOptions: this.options
    })
      .onLoad(async () => {
        this.mXComponentController.setXComponentSurfaceRect({
          surfaceWidth: previewSize.width,
          surfaceHeight: previewSize.height,
          offsetX: 0,
          offsetY: 0,
        });
        surfaceId = this.mXComponentController.getXComponentSurfaceId();
        setTimeout(async () => {                       // fixed delay - see HW-05-0043
          if (this.context) {
            await cameraShooting(cameraPosition, surfaceId, this.context.getHostContext());
          }
        }, 500);
      })
      .width(`${previewSize.width}px`)
      .height(`${previewSize.height}px`);

    Stack() {
      // 蒙版素材，要整个相机画面的，不然围绕身份证外观的蒙版一圈半透明效果实现不了。
      // ("The mask asset must cover the whole camera picture; otherwise the semi-transparent
      //   ring around the ID card outline cannot be achieved.")
      Image(this.isIDCardFront === 0 ? $r('app.media.front_photo_frame') : $r('app.media.reverse_photo_frame'))
        .objectFit(ImageFit.Fill)
        .height(400)
        .width(258);
      Text() {
        Span($r('app.string.Please01')).fontColor(Color.White);
        Span(this.isIDCardFront === 0 ? $r('app.string.ID_card_Front') : $r('app.string.ID_card_Reverse_side'))
          .fontColor($r('app.color.Font_color02'));
        Span($r('app.string.Please02')).fontColor(Color.White);
      }
      .rotate({ x: 0, y: 0, z: 1, centerX: '50%', centerY: '50%', angle: 90 })
      .translate({ x: -150, y: 0 });

      Row() {
        Image($r('app.media.warning_icon')).height(25).width(25).margin({ left: 10, right: 10 });
        Text($r('app.string.Ensure')).fontSize(16).fontColor(Color.White);
      }
      .height(36).width(320)
      .backgroundColor(Color.Black)
      .borderColor(Color.White)
      .borderRadius(18)
      .borderWidth(1)
      .rotate({ x: 0, y: 0, z: 1, centerX: '50%', centerY: '50%', angle: 90 })
      .translate({ x: 150 });
    }
    .width(`${previewSize.width}px`)
    .height(`${previewSize.height}px`);
  }
  .height(558);
}
```

Page lifecycle and shutter:

```ts
.onShown(() => {
  if (this.context) {
    cameraShooting(cameraPosition, surfaceId, this.context.getHostContext());
  }
})
.onHidden(() => {
  releaseCamera();
})
.onReady((context: NavDestinationContext) => {
  let param = context.pathInfo.param as number;
  if (param) {
    this.isIDCardFront = param;
  }
});

// shutter
Image($r('app.media.capture'))
  .onClick(async () => {
    this.isPermited = await PermissionsView.checkPermissions(this.permissions);
    if (this.isPermited) {
      capture();
      this.currentPic = true;
    } else {
      this.getUIContext().getPromptAction().showToast({
        message: $r('app.string.notPermissionMessage'),
        alignment: Alignment.Center
      });
    }
  });
```

### Profile selection

`MaskCamera.zip#entry/src/main/ets/utils/CameraShooter.ets`

```ts
let cameraOutputCap: camera.CameraOutputCapability =
  cameraManager.getSupportedOutputCapability(cameraDevices[cameraPosition], camera.SceneMode.NORMAL_PHOTO);

// pick the widest preview profile that is close to 4:3 and no wider than 1920
let previewProfile = cameraOutputCap.previewProfiles[0];
let profileWidth: number = 0;
let widhi = 0;
const EPSILON = 1e-6;
cameraOutputCap.previewProfiles.forEach((profile) => {
  widhi = profile.size.height / profile.size.width;
  if ((profile.size.height / profile.size.width - 0.75) < EPSILON) {
    if (profile.size.width >= profileWidth && profile.size.width <= 1920) {
      previewProfile = profile;
      profileWidth = profile.size.width;
    }
  }
});
imageSize = previewProfile.size;

// match the photo profile to the chosen preview size
let photoProfile = cameraOutputCap.photoProfiles[0];
cameraOutputCap.photoProfiles.forEach((profile) => {
  if (profile.size.width === imageSize.width && profile.size.height === imageSize.height) {
    photoProfile = profile;
    return;
  }
});
photoOutPut = cameraManager.createPhotoOutput(photoProfile);
```

### Session assembly

`MaskCamera.zip#entry/src/main/ets/utils/CameraShooter.ets`

```ts
cameraInput = cameraManager.createCameraInput(cameraDevices[cameraPosition]);
cameraInput.on('error', cameraDevices[cameraPosition], (error: BusinessError) => { /* ... */ });
await cameraInput.open();

previewOutput = cameraManager.createPreviewOutput(previewProfile, surfaceId);
previewOutput.on('error', (error: BusinessError) => { /* ... */ });

setPhotoOutputCb(photoOutPut);

photoSession = cameraManager.createSession<camera.PhotoSession>(camera.SceneMode.NORMAL_PHOTO);
photoSession.on('error', (error: BusinessError) => { /* ... */ });

photoSession.beginConfig();
photoSession.addInput(cameraInput);
photoSession.addOutput(previewOutput);
photoSession.addOutput(photoOutPut);
await photoSession.commitConfig();
await photoSession.start();

configuringSession(photoSession);
let zoomRatioRange = photoSession.getZoomRatioRange();
return zoomRatioRange;
```

Capability configuration, each guarded by its support check:

```ts
export function configuringSession(photoSession: camera.PhotoSession): void {
  let flashStatus: boolean = photoSession.hasFlash();
  if (flashStatus) {
    let flashModeStatus: boolean = photoSession.isFlashModeSupported(camera.FlashMode.FLASH_MODE_AUTO);
    if (flashModeStatus) {
      photoSession.setFlashMode(camera.FlashMode.FLASH_MODE_AUTO);
    }
  }
  let focusModeStatus: boolean = photoSession.isFocusModeSupported(camera.FocusMode.FOCUS_MODE_CONTINUOUS_AUTO);
  if (focusModeStatus) {
    photoSession.setFocusMode(camera.FocusMode.FOCUS_MODE_CONTINUOUS_AUTO);
  }
  let zoomRatioRange: Array<number> = photoSession.getZoomRatioRange();
  if (zoomRatioRange.length <= 0) {
    return;
  }
  photoSession.setZoomRatio(1);
}
```

### Capture and save to the media library

`MaskCamera.zip#entry/src/main/ets/utils/CameraShooter.ets`

```ts
export function capture(): void {
  let settings: camera.PhotoCaptureSetting = {
    quality: camera.QualityLevel.QUALITY_LEVEL_HIGH,
    rotation: camera.ImageRotation.ROTATION_0,
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

export function previewPhoto(context: Context | undefined): void {
  let photoContext = context as common.UIAbilityContext;
  photoContext.startAbility({
    parameters: { uri: uri },
    action: 'ohos.want.action.viewData',
    bundleName: 'com.huawei.hmos.photos',
    abilityName: 'com.huawei.hmos.photos.MainAbility'
  });
}
```

### Teardown - as shipped, and corrected

`MaskCamera.zip#entry/src/main/ets/utils/CameraShooter.ets`

```ts
// as shipped - nothing is awaited, see HW-05-0038
export async function releaseCamera(): Promise<void> {
  if (photoSession) { photoSession.stop(); }
  if (cameraInput) { cameraInput.close(); }
  if (previewOutput) { previewOutput.release(); }
  if (photoSession) { photoSession.release(); }
  if (photoOutPut) { photoOutPut.release(); }
}
```

```ts
// corrected: await each release, drop the listeners, and await the teardown at the
// top of cameraShooting before building the new pipeline
export async function releaseCamera(): Promise<void> {
  try {
    photoOutPut?.off('photoAssetAvailable');
    photoSession?.off('error');
    previewOutput?.off('error');
    if (photoSession) { await photoSession.stop(); }
    if (cameraInput) { await cameraInput.close(); }
    if (previewOutput) { await previewOutput.release(); }
    if (photoSession) { await photoSession.release(); }
    if (photoOutPut) { await photoOutPut.release(); }
  } catch (error) {
    hilog.error(0x0000, 'CameraDemo', 'release failed: %{public}s', JSON.stringify(error));
  }
}
```

## Permissions & config

`MaskCamera.zip#entry/src/main/module.json5` - as shipped:

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "pages": "$profile:main_pages",
    "requestPermissions": [
      { "name": "ohos.permission.CAMERA",         "reason": "$string:EntryAbility_desc",
        "usedScene": { "abilities": ["EntryAbility"] } },
      { "name": "ohos.permission.MICROPHONE",     "reason": "$string:EntryAbility_desc",
        "usedScene": { "abilities": ["EntryAbility"] } },
      { "name": "ohos.permission.WRITE_IMAGEVIDEO", "reason": "$string:EntryAbility_desc",
        "usedScene": { "abilities": ["EntryAbility"] } },
      { "name": "ohos.permission.READ_IMAGEVIDEO",  "reason": "$string:EntryAbility_desc",
        "usedScene": { "abilities": ["EntryAbility"] } }
    ],
    "extensionAbilities": [
      { "name": "EntryBackupAbility", "srcEntry": "./ets/entrybackupability/EntryBackupAbility.ets",
        "type": "backup", "exported": false,
        "metadata": [ { "name": "ohos.extension.backup", "resource": "$profile:backup_config" } ] }
    ]
  }
}
```

Recommended form:

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.CAMERA",
    "reason": "$string:permission_reason_camera",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
]
```

Notes on the config:

- The runtime code requests **only** `ohos.permission.CAMERA`
  (`CameraPage.ets:33-35` and `Index.ets:28-30`); the other three are declared and
  never requested (HW-05-0040).
- All four are `user_grant`, so `usedScene` is mandatory and its `when` value must
  be supplied (HW-05-0041).
- `saveCameraPhoto()` + `applyChanges` is the camera-to-gallery path, which does
  not by itself require the application to hold `WRITE_IMAGEVIDEO`; verify against
  your own device before keeping that declaration.
- `deviceTypes` is `["phone", "tablet", "2in1"]`.
- `build-profile.json5` pins `compatibleSdkVersion` / `targetSdkVersion` to
  `6.0.0(20)`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **4:3 everywhere.** The comment in the sample is explicit: the mask image must
  have the same width/height as the `XComponent` and the same 4:3 ratio, otherwise
  the dimmed ring around the card outline does not line up. The preview selection
  loop enforces the same ratio on the camera side, capped at 1920 wide.
- **The mask must cover the entire preview.** A frame-sized asset cannot produce
  the semi-transparent surround; the artwork itself carries the cut-out.
- **Landscape by rotation, not by window.** The guidance text and the warning
  strip are rotated 90 degrees and translated; the window stays portrait.
- **One pipeline per process.** `CameraShooter.ets` stores the session, input and
  outputs in module-level variables, so two capture pages cannot run at once and
  any second initialisation must release the first.
- **Camera capabilities are device-dependent.** Flash, auto flash mode,
  continuous auto focus and the zoom range are all probed before use; do not
  assume any of them exist.
- **The Gallery jump is hard-coded.** `previewPhoto` starts
  `com.huawei.hmos.photos` / `com.huawei.hmos.photos.MainAbility` by name, so it
  only works where that system app is present.
- **`imageAIOptions` is attached but unused.** The `XComponent` receives an
  `ImageAIOptions` with `SUBJECT` and `TEXT` analyzers and an
  `ImageAnalyzerController`, but no analysis is ever started in this sample.

## Pitfalls

- **`hilog.error(-1, ...)` is incorrect.** The domain must be within
  `[0x0, 0xFFFF]`, and the reference states that outside that range "logs cannot
  be printed" - so every camera-initialisation failure in `CameraShooter.ets` is
  invisible. The same calls also pass the message as the `tag` and an empty string
  as the `format`. (HW-05-0037)
- **Calling `releaseCamera()` without `await`, and not awaiting the five releases
  inside it, is incorrect.** `cameraShooting` starts opening a new input while the
  old one is still closing, and `onHidden` followed by a quick `onShown` overlaps
  teardown with rebuild. (HW-05-0038)
- **Registering `on('error')` three times and `on('photoAssetAvailable')` once per
  `cameraShooting` call with no `off(...)` is incorrect** - and the page calls
  `cameraShooting` from both `onLoad` and `onShown`, so the asset callback can run
  more than once per shutter press. (HW-05-0039)
- **The document's 权限说明 listing only `ohos.permission.CAMERA` is incorrect
  relative to the sample,** which declares four user_grant permissions while
  requesting one. Reconcile the two. (HW-05-0040)
- **`usedScene` without `when` is incorrect** for user_grant permissions; add
  `"when": "inuse"`. (HW-05-0041)
- **`requestPermissionOnSetting(context.getHostContext(), ...)` is incorrect** -
  `getHostContext()` is `Context | undefined` and the parameter is a mandatory
  `Context`; cast it as the same class already does one method earlier.
  (HW-05-0042)
- **Starting the camera from `onShown` with the module-level `surfaceId` is
  incorrect**: that variable is `''` until the `XComponent` `onLoad` assigns it,
  and the 500 ms `setTimeout` in `onLoad` is itself a symptom of the same
  ordering problem. Guard on a non-empty surface id and initialise from one place.
  (HW-05-0043)
- **The document's mask snippet does not compile, which is incorrect.**
  `ID_card` / `ID_card_reverseSide` are undeclared, the field is spelled
  `isIDcardFront` instead of the sample's `isIDCardFront`, and a media resource is
  used as `Span` content. Use `$r('app.media.front_photo_frame')` /
  `$r('app.media.reverse_photo_frame')` for the `Image` and
  `$r('app.string.ID_card_Front')` / `$r('app.string.ID_card_Reverse_side')` for
  the `Span`. (HW-05-0044)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/03_system/js-apis-hilog.md` - the
  `[0x0, 0xFFFF]` domain range, "If the value exceeds the range, logs cannot be
  printed", and the `(domain, tag, format, ...args)` signature.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
- `documentation/harmonyos-references/02_application-framework/js-apis-abilityaccessctrl.md` -
  `requestPermissionOnSetting12+` with its mandatory `Context` parameter.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-abilityaccessctrl
- `documentation/harmonyos-references/02_application-framework/js-apis-permissionrequestresult.md` -
  `dialogShownResults` semantics used by the permission ladder.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-permissionrequestresult
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` -
  `getHostContext(): Context | undefined`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `usedScene`
  being mandatory for user_grant permissions and its `when` parameter.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `documentation/harmonyos-guides/04_system/request-user-authorization-second.md` -
  the cast used when passing the host context to `requestPermissionOnSetting`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization-second
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-mediaassetchangerequest.md` -
  `MediaAssetChangeRequest` and `saveCameraPhoto`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-mediaassetchangerequest
