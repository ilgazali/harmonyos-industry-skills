---
id: UTIL-17
title: VIN scanner - dual-channel camera preview with per-frame OCR and a regex gate
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/17_vehicle_frame_number_identification.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/vehicle_frame_number_identification-0000002294949328
sample: huawei_industry_tree/15_utilities/downloads/ScanFrameNo.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CameraKit", "@kit.CoreFileKit", "@kit.CoreVisionKit", "@kit.ImageKit", "@kit.PerformanceAnalysisKit"]
apis: [abilityAccessCtrl, camera, common, display, hilog, image, textRecognition, window]
permissions: [ohos.permission.CAMERA]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0042, HW-15-0043, HW-15-0044, HW-15-0045, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when you need to **read a fixed-format code off a live camera
preview and stop as soon as it is found** - a vehicle identification number
here, but the same shape covers a container number, a meter serial, an IMEI,
a policy number on a paper form. The user never presses a shutter: the app
watches every frame, runs OCR on it, and the first frame whose text matches
the expected pattern ends the scan.

The load-bearing technique is **dual-channel preview**. One `PreviewOutput`
is bound to the `XComponent` surface the user sees; a second `PreviewOutput`
is bound to the surface of an `image.ImageReceiver`, and the recognition
pipeline drains that second channel. The displayed preview never stutters
because nothing in the OCR path touches it, and the recogniser gets
full-resolution YUV frames rather than a screenshot.

Two things generalise beyond the demo. First, the **format gate is the whole
feature**: OCR on a moving preview returns noise constantly, so a scanner is
only as good as the regular expression (or checksum) that decides which
result to accept. Second, a camera pipeline built this way has **two objects
that must be torn down, not one** - the session/input/outputs and the
`ImageReceiver` - and this sample gets the second half wrong
(`HW-15-0044`). Read the pitfalls before copying `CameraService.ets`.

## Feature checklist

- Home page shows a VIN text field and a 查询车辆信息 (query vehicle
  information) button; the scan icon in the title bar opens the camera page.
- Tapping the scan icon requests `ohos.permission.CAMERA` first and toasts an
  error if the user refuses.
- The camera page shows a live preview with a white capture-guide rectangle
  and a flash toggle.
- While the preview runs, every arriving frame is OCR'd until one produces a
  17-character string matching the VIN pattern.
- On a match the scan stops immediately (the `imageArrival` listener is
  removed), the recognised region of the frame is cropped out as a `PixelMap`,
  and both the string and the crop are published to `AppStorage`.
- The camera page pops itself; the home page shows the cropped VIN image plus
  a vehicle-information card.
- Typing a VIN by hand and pressing the query button shows the same card
  (broken as shipped - `HW-15-0042`).
- Leaving the camera page releases the camera; returning re-initialises it.

## Architecture

One `entry` module. The camera is a singleton service; the two pages
communicate only through `AppStorage` and a global map.

```
entry/src/main/ets
├── common/Constants.ets            numeric literals: 17 (VIN length), 1920x1080 profile, -1 sentinel
├── entryability/EntryAbility.ets   loads pages/Index, forces light colour mode
├── entrybackupability/
├── model/VehicleInfo.ets           the result DTO (brand, carType, modelYear, ...)
├── pages/Index.ets                 @Entry: Navigation host, VIN input, result card, scan button
├── pages/ScanPage.ets              NavDestination: XComponent preview, flash toggle, guide frame
├── service/CameraService.ets       the whole camera + OCR pipeline, exported as a singleton
└── utils/GlobalContext.ets         a string->Object map + the UIAbilityContext the camera needs
    utils/Logger.ets
```

The documented 工程目录 matches the zip file for file.

**The design decision worth copying** is the split between the two preview
channels and the way the result crosses back. `CameraService` knows nothing
about either page: it writes `vinNumber` and `frameNoImage` into `AppStorage`,
and both pages hold `@Watch('onScanResult') @StorageProp('vinNumber')`.
`ScanPage`'s watcher pops the navigation stack; `Index`'s watcher fills in the
result. One write, two independent reactions, no callback plumbed through the
`NavPathStack`. That is the right coupling for a scanner that may be entered
from several places.

**The decision worth avoiding** sits right next to it: `CameraService` is
`export default new CameraService()` and `ScanPage` keeps a module-level
`let isInit = false`. Both survive page destruction, so the correctness of
re-entry depends entirely on `releaseCamera()` undoing everything
`initCamera()` did - and it does not undo the `ImageReceiver`
(`HW-15-0044`). If you copy this file, make the service instance-scoped or
make the teardown symmetric.

## Implementation steps

1. **Request `ohos.permission.CAMERA` at the tap**, not at launch, and only
   push the scan route when `authResults[0] === 0`. Clear `vinNumber` and
   `frameNoImage` in `AppStorage` before pushing so a stale result does not
   pop the page instantly.
2. **Hand the `XComponent` surface id to the service from
   `onSurfaceCreated`**, and re-initialise from `NavDestination.onShown` when
   the surface already exists but the service was released.
3. **Pick the preview profile from the device**, never from the literal:
   `getSupportedOutputCapability(...).previewProfiles` is searched for
   1920x1080 `CAMERA_FORMAT_YUV_420_SP` and falls back to `previewProfiles[0]`
   when the search returns `-1`.
4. **Create two `PreviewOutput`s from the same profile** - one on the
   XComponent surface, one on `imageReceiver.getReceivingSurfaceId()` - and
   `addOutput` both before `commitConfig()`.
5. **Register `imageArrival` and read frames with `readNextImage`.** Release
   every `image.Image` on every path, including the error paths, or the
   8-slot buffer starves (`HW-15-0044`).
6. **Guard the OCR call with a global "already scanned" flag** so the frames
   still in flight when a match lands do not start a second recognition.
7. **Gate the recognised text on a real VIN pattern.** The shipped class is
   corrupted and rejects whole manufacturer prefixes (`HW-15-0043`).
8. **Crop with the returned corner points**, not with a fixed rectangle:
   `data.blocks[0].lines[0].cornerPoints` gives the four corners of the
   recognised line, and `pixelMap.cropSync` turns them into the thumbnail the
   result page shows.
9. **Tear the pipeline down symmetrically on `onWillHide`/`onHidden`**:
   outputs, session, input *and* `imageReceiver.off('imageArrival')` +
   `imageReceiver.release()` (`HW-15-0044`).
10. **Branch on the flash capability queries** before calling `setFlashMode`
    (`HW-15-0045`).
11. **Bind the manual VIN field two-way** so the typed value reaches state
    (`HW-15-0042`).

## Verified snippets

All snippets are from `ScanFrameNo.zip`. Corrected forms are marked.

**Dual-channel preview - `entry/src/main/ets/service/CameraService.ets`** (as shipped)

```typescript
// Create previewOutput output object.
this.previewOutput = this.createPreviewOutputFn(this.cameraManager, this.previewProfileObj, surfaceId);
if (this.previewOutput === undefined) {
  Logger.error(TAG, 'Failed to create the preview stream.');
  return;
}
// Monitor preview events.
this.previewOutputCallBack(this.previewOutput);

// Create imageReceiver output object.
this.imageReceiver = image.createImageReceiver(this.previewProfileObj.size, image.ImageFormat.JPEG, 8);
// 获取第一路流SurfaceId。
let imageReceiverSurfaceId = await this.imageReceiver.getReceivingSurfaceId();
this.imageReceiverOutput =
  this.createPreviewOutputFn(this.cameraManager, this.previewProfileObj, imageReceiverSurfaceId);
if (this.imageReceiverOutput === undefined) {
  Logger.error(TAG, 'Failed to create the preview stream.');
  return;
}
this.imageReceiverCallBack(this.imageReceiver);
this.previewOutputCallBack(this.imageReceiverOutput);
```

and the session that joins them:

```typescript
this.session.beginConfig();
this.session.addInput(cameraInput);
this.session.addOutput(previewOutput);          // the XComponent surface
this.session.addOutput(imageReceiverOutput);    // the ImageReceiver surface
await this.session.commitConfig();
this.setFocusMode(camera.FocusMode.FOCUS_MODE_CONTINUOUS_AUTO);
await this.session.start();
```

**Three details carry this design.** Both outputs are built from the *same*
`previewProfileObj`, which is mandatory - a session rejects two preview
streams whose resolutions the device cannot run concurrently, and reusing the
negotiated profile is the cheapest way to stay legal. The receiver is created
with a capacity of `8`: that is the number of frames that may be outstanding
before delivery stalls, and it is exactly why an un-released `image.Image` on
an error path is fatal rather than untidy. And `FOCUS_MODE_CONTINUOUS_AUTO`
is set *after* `commitConfig()` and before `start()` - focus modes are
session-scoped, so setting them earlier throws.

Note that the second channel is declared as a `PreviewOutput`, not a
`PhotoOutput`. There is no shutter in this feature; the recogniser simply
consumes the preview stream a second time.

**Frame handling and OCR - same file** (corrected, see `HW-15-0044`, `HW-15-0043`)

```typescript
// FIX: [A-HJ-NPR-Z1-9] first char - the shipped class is '[WFV JensGHLRSTXYZ123456789]'
const REGEXP = new RegExp('^[A-HJ-NPR-Z1-9]{1}[A-HJ-NPR-Z0-9]{16}$');

imageReceiverCallBack(receiver: image.ImageReceiver): void {
  receiver.on('imageArrival', () => {
    receiver.readNextImage(async (err, nextImage: image.Image) => {
      if (err || nextImage === undefined) {
        Logger.error(TAG, `imageArrival error, error is ${JSON.stringify(err)}`);
        return;
      }
      if (GlobalContext.get().getT<boolean>('hasScanResult')) {
        nextImage.release();                      // FIX: shipped code drops the frame unreleased
        return;
      }
      this.recognizeText(nextImage);
    });
  });
}

recognizeText(nextImage: image.Image) {
  nextImage.getComponent(image.ComponentType.JPEG, async (err, imgComponent: image.Component) => {
    try {
      if (err || imgComponent === undefined || !(imgComponent.byteBuffer as ArrayBuffer)) {
        return;                                   // FIX: shipped code returns before release()
      }
      let sourceOptions: image.SourceOptions = {
        sourceDensity: 0,
        sourcePixelFormat: image.PixelMapFormat.NV21,
        sourceSize: this.previewProfileObj.size
      };
      let imageSource: image.ImageSource = image.createImageSource(imgComponent.byteBuffer, sourceOptions);
      let opts: image.DecodingOptions = {
        editable: false,
        desiredPixelFormat: image.PixelMapFormat.NV21,
        desiredSize: this.previewProfileObj.size,
        rotate: 90.0
      };
      let pixelMap = imageSource.createPixelMapSync(opts);
      await imageSource.release();
      // ... recognizeText(visionInfo, textConfiguration) unchanged - see the next snippet
    } finally {
      nextImage.release();                        // FIX: the only release, on every path
    }
  });
}
```

**The buffer is the whole story here.** `createImageReceiver(size, JPEG, 8)`
hands out at most eight `image.Image` handles; `readNextImage` takes one and
only `release()` gives it back. The shipped code releases at the bottom of
`getComponent`'s callback, which three separate `return`s skip - a run of
frames where `getComponent` errors, or a handful of frames arriving after a
successful scan, permanently consumes slots until the preview simply stops
delivering. Wrapping the body in `try/finally` is the smallest change that
makes the invariant unconditional.

The decode options are worth reading twice: the receiver format is `JPEG`
but the *source* and *desired* pixel formats are `NV21`, because the JPEG
component of a YUV preview frame carries NV21 planes. `rotate: 90.0` is
applied at decode time so the recogniser sees an upright image on a
portrait-held phone - cheaper than rotating the `PixelMap` afterwards.

**The format gate - same file** (corrected, see `HW-15-0043`)

```typescript
textRecognition.recognizeText(visionInfo, textConfiguration)
  .then(async (data: textRecognition.TextRecognitionResult) => {
    if (!data || !data.value || data.value.length < Constants.FRAME_NO_LENGTH ||
      GlobalContext.get().getT<boolean>('hasScanResult')) {
      return;
    }
    let recognitionString = data.value.replaceAll('*', '');
    if (REGEXP.test(recognitionString) && !GlobalContext.get().getT<boolean>('hasScanResult')) {
      GlobalContext.get().setObject('hasScanResult', true);
      this.imageReceiver?.off('imageArrival');
      // 截取目标区域
      let cornerPoint = data.blocks[0].lines[0].cornerPoints;
      let pointx: number = Math.min(cornerPoint[0].x, cornerPoint[3].x);
      let pointy: number = Math.min(cornerPoint[0].y, cornerPoint[1].y);
      let width: number =
        Math.max(cornerPoint[1].x, cornerPoint[2].x) - Math.min(cornerPoint[0].x, cornerPoint[3].x);
      let height: number =
        Math.max(cornerPoint[2].y, cornerPoint[3].y) - Math.min(cornerPoint[0].y, cornerPoint[1].y);
      pixelMap.cropSync({ size: { width: width, height: height }, x: pointx, y: pointy });
      AppStorage.setOrCreate('frameNoImage', pixelMap);
      AppStorage.setOrCreate('vinNumber', recognitionString);
    }
  }).catch((error: BusinessError) => {
    Logger.error(TAG, `Failed to recognize text. Code: ${error.code}, message: ${error.message}`);
  });
```

**`hasScanResult` is checked three times and that is not paranoia.** OCR is
asynchronous and several frames are in flight at once, so between the length
check and the regex test another frame's promise can already have won.
Setting the flag *before* `off('imageArrival')` closes the last window: the
listener removal only stops new frames, it does nothing about the ones
already inside `recognizeText`.

The real VIN alphabet is `A-Z0-9` minus `I`, `O`, `Q` (they are excluded by
ISO 3779 to avoid confusion with 1 and 0) and, in the first position, minus
`0`. `[A-HJ-NPR-Z1-9]` expresses exactly that. The shipped class
`[WFV JensGHLRSTXYZ123456789]` reads like a hand-typed list of European WMI
prefixes that was then mangled: it contains a literal space and the lowercase
run `ens`, and it omits `A`, `B`, `C`, `D`, `E`, `K`, `M`, `N`, `P` and `U` -
so every Hyundai (`KMH…`), Kia (`KNA…`), Nissan built in Japan and most
Chinese-market VINs are silently invisible to the scanner.

**Two-way binding on the manual field - `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-15-0042`)

```typescript
TextInput({ text: $$this.frameNo, placeholder: $r('app.string.enter_the_frame_number') })
  .placeholderFont({ size: $r('app.float.font_size_16'), weight: FontWeight.Normal })
  .width('80%')
  .height($r('app.float.height_40'))
  .margin({ left: $r('app.float.margin_2') });
```

```typescript
Button($r('app.string.querying_vehicle_information'))
  .onClick(() => {
    if (this.frameNo) {              // stays false forever with the shipped one-way binding
      this.vehicleInfo = VEHICLE_INFO;
      this.showResult = true;
    }
  });
```

`TextInput({ text: this.frameNo })` is a **one-way** initialisation: state
flows into the component, never back. The shipped code has no `onChange`
either, so `this.frameNo` is `''` for the entire life of the page unless a
scan writes it, and the query button's guard can never pass. `$$` is the
built-in two-way binding sugar for exactly this case; an explicit
`.onChange((v: string) => { this.frameNo = v; })` is equivalent and clearer
if you also need to normalise or upper-case the input.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.CAMERA",
    "reason": "$string:reason_camera",
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": "always"
    }
  }
]
```

- `ohos.permission.CAMERA` is `user_grant`, so `reason` and `usedScene` are
  mandatory and the reason string resource must exist.
- `"when": "always"` is over-broad for this feature. The camera is used only
  while the scan page is foreground; `"inuse"` describes it accurately and is
  what a reviewer will expect.
- The request is raised at the scan tap in `Index.titleBar()`, which is the
  right moment - the home page is fully usable without the camera.
- `deviceTypes` is `phone`, `tablet`, `2in1`. There is no wearable in the
  list, which matters because the flash and continuous-focus calls have no
  guarded fallback.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- The vehicle data is a hardcoded `VEHICLE_INFO` constant. Whatever VIN is
  scanned or typed, the same 上海 / XX SUV / 2024 card appears - this is a
  scanning sample, not a lookup service.
- `previewProfileObj` starts at 1920x1080; devices that offer no matching
  profile silently fall back to `previewProfiles[0]`, which may have a very
  different aspect ratio from the `XComponent` surface.
- The white guide rectangle is decorative. Recognition runs on the whole
  frame, not on the region inside it, so a VIN outside the box scans just as
  well and text elsewhere in the frame competes for the match.
- `CameraService` is a module singleton and `isInit` is a module-level
  variable in `ScanPage.ets`; neither is reset when the page is destroyed.
- OCR is `textRecognition` with `isDirectionDetectionSupported: false`, so a
  sideways VIN plate will not be recognised.

## Pitfalls

- **`HW-15-0042`** (B/medium, confirmed): the manual VIN flow is completely
  dead - `TextInput({ text: this.frameNo })` is one-way and there is no
  `onChange`, so `if (this.frameNo)` in the query button never passes, and
  after a scan the button is replaced by the result view so the path can never
  run at all. Fix: bind with `$$this.frameNo` or add an `onChange` handler.
- **`HW-15-0043`** (B/medium, confirmed): the VIN regex first-character class
  `[WFV JensGHLRSTXYZ123456789]` contains a literal space and the lowercase
  run `ens` while omitting `A`,`B`,`C`,`D`,`E`,`K`,`M`,`N`,`P`,`U`, so whole
  manufacturer families (`KMH…`, `KNA…`, `MA…`) never match while a
  leading space or lowercase letter passes. Fix: use `[A-HJ-NPR-Z1-9]` for the
  first character and `[A-HJ-NPR-Z0-9]{16}` for the rest.
- **`HW-15-0044`** (B/medium, confirmed): **systematic across this industry** -
  `ImageReceiver` pipelines leak. `releaseCamera()` releases the outputs, the
  session and the input but never calls `off('imageArrival')` or
  `release()` on the receiver, so every re-entry to the scan page stacks
  another receiver; and the error paths in `readNextImage`/`getComponent`
  return without `nextImage.release()`, starving the 8-slot buffer until frame
  delivery stops. The same defect appears in `UTIL-41` (`ImageDecoder`
  `PreviewPage.ets`), which has no teardown at all. Fix: `off` + `release`
  the receiver in the release path, and release `nextImage` in a `finally`.
- **`HW-15-0045`** (B/low, confirmed): `hasFlashFn` queries `hasFlash()` and
  `isFlashModeSupported(flashMode)`, logs both results, and then calls
  `setFlashMode(flashMode)` unconditionally. On a flashless device the call
  throws instead of being skipped. Fix: return early when either query is
  false. Compare `setFocusMode` in the same file, which does branch correctly.

## References

- `documentation/harmonyos-guides/02_media/camera-kit.md` - session lifecycle,
  `beginConfig`/`addOutput`/`commitConfig`/`start`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-kit
- `documentation/harmonyos-guides/02_media/camera-preview.md` - preview
  streams and surface binding
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-preview
- `documentation/harmonyos-references/04_media/js-apis-camera.md` -
  `PreviewOutput`, `FlashMode`, `FocusMode`, `SceneMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-camera
- `documentation/harmonyos-references/04_media/js-apis-image.md` -
  `createImageReceiver`, `readNextImage`, `Image.release`, `getComponent`,
  `DecodingOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-image
- `documentation/harmonyos-guides/08_ai/core-vision-introduction.md` - Core
  Vision Kit overview
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/core-vision-introduction
- `documentation/harmonyos-guides/08_ai/core-vision-text-recognition.md` -
  `recognizeText`, `VisionInfo`, `cornerPoints`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/core-vision-text-recognition
- `documentation/harmonyos-guides/04_system/permissions-for-all-user.md` -
  `ohos.permission.CAMERA`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all-user
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` -
  the `user_grant` request flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-textinput.md`
  and `documentation/harmonyos-guides/03_application-framework/arkts-two-way-sync.md`
  - `$$` two-way binding for `TextInput`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-textinput
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-two-way-sync
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md`
  - `NavPathStack`, `NavDestination`, `routerMap`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `UTIL-41` - the second instance of the `ImageReceiver` leak
