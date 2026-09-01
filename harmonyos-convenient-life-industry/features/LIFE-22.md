---
id: LIFE-22
title: Scan an ID card with a live custom camera - dual-channel preview into Core Vision OCR and Natural Language entity extraction
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/22_id_card_scan.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/id_card_scan-0000002338766518
sample: huawei_industry_tree/02_convenient_life/downloads/ScanIdCard.zip
kits: ["@kit.CameraKit", "@kit.ImageKit", "@kit.CoreVisionKit", "@kit.NaturalLanguageKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.ArkTS", "@kit.PerformanceAnalysisKit"]
apis: ["camera.getCameraManager", "cameraManager.getSupportedCameras", "cameraManager.getSupportedSceneModes", "cameraManager.getSupportedOutputCapability", "cameraManager.createPreviewOutput", "cameraManager.createCameraInput", "cameraManager.createSession", "cameraManager.on('cameraStatus')", "cameraManager.off('cameraStatus')", "cameraInput.open", "cameraInput.close", "cameraInput.on('error')", "session.beginConfig", "session.addInput", "session.addOutput", "session.commitConfig", "session.start", "session.release", "session.on('error')", "session.isFocusModeSupported", "session.setFocusMode", "previewOutput.on('frameStart')", "previewOutput.on('frameEnd')", "previewOutput.on('error')", "previewOutput.release", "image.createImageReceiver", "imageReceiver.getReceivingSurfaceId", "imageReceiver.on('imageArrival')", "imageReceiver.off('imageArrival')", "imageReceiver.readNextImage", "nextImage.getComponent", "nextImage.release", "image.createImageSource", "imageSource.createPixelMapSync", "imageSource.release", "textRecognition.recognizeText", "textProcessing.getEntity", EntityType, XComponent, XComponentController, "controller.setXComponentSurfaceRect", "controller.onSurfaceCreated", "controller.onSurfaceDestroyed", Navigation, NavPathStack, NavDestination, "abilityAccessCtrl.createAtManager", "atManager.requestPermissionsFromUser", "AppStorage.setOrCreate", StorageProp, Watch, "display.getDefaultDisplaySync"]
permissions: ["ohos.permission.CAMERA"]
min_api: 20
modules: [entry]
findings: [HW-02-0158, HW-02-0159, HW-02-0160, HW-02-0161, HW-02-0162, HW-02-0163, HW-02-0164, HW-02-0165, HW-02-0166, HW-02-0167, HW-02-0168, HW-02-0169, HW-02-0170, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card for **live scanning into a form**: the camera stays open, every
preview frame is inspected, and the moment the target document is in frame the
page pops itself and the fields are already filled. Real-name registration with
an ID card is the case the sample builds, but the shape is the same for any
document you want recognised without the user pressing a shutter.

The distinguishing move is **dual-channel preview**. One preview stream drives
the visible surface; a second preview stream is bound to an `ImageReceiver`, and
that one exists purely so the application can read pixels:

```
XComponent (SURFACE)  --surfaceId-->  previewOutput        the picture the user sees
ImageReceiver         --surfaceId-->  imageReceiverOutput  the pixels the app reads
                          |
                          v
        on('imageArrival') -> readNextImage -> getComponent(JPEG)
                          -> createImageSource -> createPixelMapSync
                          -> textRecognition.recognizeText   Core Vision Kit
                          -> textProcessing.getEntity        Natural Language Kit
                          -> AppStorage['idCardModel'] -> @StorageProp -> the form
```

Two preview outputs are added to one session, so this is not "take a photo and
decode it" - there is no PhotoOutput anywhere in the sample.

**Take a different card if you do not need a live preview.** `LIFE-21` does the
same OCR plus entity extraction from a picked or captured still image, with
`PhotoViewPicker` / `cameraPicker.pick` and **no camera permission at all**. The
whole cost of this card - the CAMERA permission, the session lifecycle, the
per-frame buffer discipline below - buys you only the automatic capture. If a
shutter press is acceptable, `LIFE-21` is the cheaper and safer feature.

`LIFE-19` covers `textProcessing.getEntity` on its own.

## Feature checklist

- [ ] `ohos.permission.CAMERA` declared in `module.json5` and requested at
      runtime before the scan page is pushed.
- [ ] An `XComponent` of type `SURFACE` with a controller subclass, using the
      **`XComponentOptions` overload** - not the deprecated id-based one
      (HW-02-0158).
- [ ] The surface id taken from `onSurfaceCreated`, not from
      `getXComponentSurfaceId()` in `onLoad` (HW-02-0168).
- [ ] Two preview outputs on one session: the visible surface and an
      `ImageReceiver` surface.
- [ ] `imgComponent.rowStride` compared with the frame width before the buffer
      is decoded (HW-02-0160).
- [ ] `nextImage.release()` on **every** exit path of the frame callback
      (HW-02-0161).
- [ ] `pixelMap.release()` after the recognition promise settles (HW-02-0159).
- [ ] One recognition in flight at a time (HW-02-0165).
- [ ] Teardown releases the session, both outputs, the camera input **and the
      `ImageReceiver`**, and unsubscribes `imageArrival` (HW-02-0162).
- [ ] Teardown wired to exactly one lifecycle callback (HW-02-0164).
- [ ] No card text, entity list, name or ID number written to the log
      (HW-02-0163).

## Architecture

Three files carry the feature; the rest is a form.

| File | Role |
| --- | --- |
| `pages/MainPage.ets` | `@Entry`. Hosts the `Navigation` stack, requests CAMERA, pushes `ScanPage`, and renders the two `TextInput` fields bound to `@StorageProp('idCardModel')`. |
| `pages/ScanPage.ets` | The `NavDestination`. Owns the `XComponent` and the controller subclass, and drives `CameraService` from the navigation lifecycle. |
| `service/CameraService.ets` | A module-level singleton (`export default new CameraService()`). Owns the camera manager, input, session, both preview outputs, the `ImageReceiver`, and the whole recognition pipeline. |

Result transport is `AppStorage`, not the navigation stack. `CameraService`
writes `AppStorage.setOrCreate('idCardModel', model)` from deep inside the
recognition callback; both pages hold
`@Watch('onScanResult') @StorageProp('idCardModel')`, so `ScanPage` pops itself
and `MainPage` fills its fields from the same write. That is why the service
does not need a reference to either page.

`GlobalContext` is a hand-rolled `Map<string, Object>` singleton carrying the
`UIAbilityContext` (for `camera.getCameraManager`), the surface id, and the
`hasScanResult` flag that stops the pipeline once a card has been read.

The scan page is registered by `routerMap`, not by `main_pages.json`:

```json
// entry/src/main/resources/base/profile/router_map.json
{
  "routerMap": [
    {
      "name": "ScanPage",
      "pageSourceFile": "src/main/ets/pages/ScanPage.ets",
      "buildFunction": "scanPageBuilder",
      "data": { "description": "this is ScanPage" }
    }
  ]
}
```

with `"routerMap": "$profile:router_map"` in `module.json5` and
`"pages": "$profile:main_pages"` listing only `pages/MainPage`.

## Implementation steps

Where the shipped code is wrong, the step below gives the corrected version and
names the finding.

1. **Declare the permission.** One user-grant permission, with a reason string:

   ```json5
   "requestPermissions": [
     {
       "name": "ohos.permission.CAMERA",
       "reason": "$string:reason_camera",
       "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
     }
   ]
   ```

2. **Request it before navigating.** Push the scan page only from inside the
   grant branch, and reset the result state on the way in:

   ```ts
   let atManager = abilityAccessCtrl.createAtManager();
   atManager.requestPermissionsFromUser(this.getUIContext().getHostContext(), ['ohos.permission.CAMERA'])
     .then((result: PermissionRequestResult): void => {
       if (result.authResults[0] === 0) {
         GlobalContext.get().setObject('hasScanResult', false);
         AppStorage.setOrCreate('idCardModel', undefined);
         this.pathStack.pushPathByName('ScanPage', null);
       } else {
         this.getUIContext().getPromptAction().showToast({ message: $r('app.string.permission_denied') });
       }
     })
   ```

   `authResults[0] === 0` is the grant. Route the denial message through a
   resource string rather than a literal (HW-02-0167).

3. **Subclass the controller and start the camera from `onSurfaceCreated`.**
   This is the step the document gets wrong (HW-02-0168):

   ```ts
   class MyXComponentController extends XComponentController {
     onSurfaceCreated(surfaceId: string): void {
       GlobalContext.get().setObject('xComponentSurfaceId', surfaceId);
       isInit = true;
       CameraService.initCamera(surfaceId);
     }

     onSurfaceDestroyed(surfaceId: string): void {
       GlobalContext.get().setObject('xComponentSurfaceId', '');
     }
   }
   ```

   The component reference is explicit that `onLoad` fires **before**
   `onSurfaceCreated`, so a `getXComponentSurfaceId()` call in `onLoad` may not
   return a valid id.

4. **Declare the `XComponent` with the current overload** (HW-02-0158) and use
   `onLoad` only for the surface rectangle:

   ```ts
   XComponent({
     type: XComponentType.SURFACE,
     controller: this.mXComponentController
   })
     .width('100%')
     .height('100%')
     .onLoad(() => {
       let screen = display.getDefaultDisplaySync();
       this.mXComponentController.setXComponentSurfaceRect({
         surfaceWidth: screen.width,
         surfaceHeight: screen.height
       });
     })
   ```

5. **Pick the camera, the scene mode and the preview profile.** All three
   lookups can fail, and each failure must stop the flow:

   ```ts
   this.cameraManager = camera.getCameraManager(GlobalContext.get().getCameraSettingContext());
   this.cameras = this.cameraManager.getSupportedCameras();
   this.curCameraDevice = this.cameras[this.defaultCameraDeviceIndex];
   if (!this.isSupportedSceneMode(this.cameraManager, this.curCameraDevice)) {
     return;
   }
   let capability = this.cameraManager.getSupportedOutputCapability(this.curCameraDevice, this.curSceneMode);
   let previewProfile = this.getPreviewProfile(capability);  // falls back to previewProfiles[0]
   ```

   `getPreviewProfile` looks for the requested 1920x1080 profile and falls back
   to the first supported one, so the requested resolution is a preference, not
   a promise. Everything downstream must read `this.previewProfileObj` after the
   assignment, never the literal.

6. **Create both preview outputs.** The first is bound to the visible surface,
   the second to the `ImageReceiver`:

   ```ts
   this.previewOutput = this.createPreviewOutputFn(this.cameraManager, this.previewProfileObj, surfaceId);

   this.imageReceiver = image.createImageReceiver(this.previewProfileObj.size, image.ImageFormat.JPEG, 8);
   let imageReceiverSurfaceId = await this.imageReceiver.getReceivingSurfaceId();
   this.imageReceiverOutput =
     this.createPreviewOutputFn(this.cameraManager, this.previewProfileObj, imageReceiverSurfaceId);
   ```

   The size and format handed to `createImageReceiver` are inert - the guide
   states the parameters "do not have a practical impact" and that the image
   properties come from the producer's profile. The capacity, 8, is not inert:
   it is the number of buffers you may hold at once, which is what step 8 is
   about.

7. **Commit the session with both outputs.**

   ```ts
   this.session = cameraManager.createSession(this.curSceneMode) as camera.PhotoSession;
   this.session.beginConfig();
   this.session.addInput(cameraInput);
   this.session.addOutput(previewOutput);
   this.session.addOutput(imageReceiverOutput);
   await this.session.commitConfig();
   this.setFocusMode(camera.FocusMode.FOCUS_MODE_CONTINUOUS_AUTO);
   await this.session.start();
   ```

   Continuous auto focus is what makes hands-free scanning work; without it the
   frames the OCR sees stay soft.

8. **Read frames with the buffer discipline the guide requires.** Corrected for
   HW-02-0160, HW-02-0161, HW-02-0159 and HW-02-0165:

   ```ts
   private isRecognizing = false;

   imageReceiverCallBack(receiver: image.ImageReceiver): void {
     receiver.on('imageArrival', () => {
       receiver.readNextImage((err, nextImage: image.Image) => {
         if (err || nextImage === undefined) {
           return;
         }
         if (GlobalContext.get().getT<boolean>('hasScanResult') || this.isRecognizing) {
           nextImage.release();          // still owed back to the receiver
           return;
         }
         this.isRecognizing = true;
         this.recognizeText(nextImage);
       });
     });
   }

   recognizeText(nextImage: image.Image) {
     nextImage.getComponent(image.ComponentType.JPEG, (err, imgComponent: image.Component) => {
       if (err || imgComponent === undefined || !imgComponent.byteBuffer) {
         nextImage.release();
         this.isRecognizing = false;
         return;
       }
       const width = this.previewProfileObj.size.width;
       const height = this.previewProfileObj.size.height;
       const stride = imgComponent.rowStride;
       let buffer = imgComponent.byteBuffer;
       if (stride !== width) {
         const dstArr = new Uint8Array(width * height * 1.5);
         for (let j = 0; j < height * 1.5; j++) {
           dstArr.set(new Uint8Array(imgComponent.byteBuffer, j * stride, width), j * width);
         }
         buffer = dstArr.buffer;
       }
       let imageSource = image.createImageSource(buffer, {
         sourceDensity: 0,
         sourcePixelFormat: image.PixelMapFormat.NV21,
         sourceSize: this.previewProfileObj.size
       });
       let pixelMap = imageSource.createPixelMapSync({
         editable: false,
         desiredPixelFormat: image.PixelMapFormat.NV21,
         desiredSize: this.previewProfileObj.size,
         rotate: 90.0
       });
       imageSource.release();
       nextImage.release();              // the buffer has been copied out

       textRecognition.recognizeText({ pixelMap: pixelMap },
         { isDirectionDetectionSupported: false })
         .then((data: textRecognition.TextRecognitionResult) => this.handleText(data))
         .catch((error: BusinessError) => {
           Logger.error(TAG, `recognizeText failed. code: ${error.code}`);
         })
         .finally(() => {
           pixelMap.release();
           this.isRecognizing = false;
         });
     });
   }
   ```

   Four rules in one block: the stride is checked, `nextImage` is released on
   every path, the `PixelMap` is released when the recognition settles, and only
   one recognition runs at a time.

9. **Narrow the recognised text before entity extraction.** The card layout is
   used as a filter, so the OCR of the whole card is cut down to the two regions
   that carry the name and the number:

   ```ts
   let firstIndex = data.value.indexOf('姓名');            // "name"
   let startIndex = data.value.indexOf('性别');            // "sex"
   let endIndex = data.value.indexOf('\n公民身份号码');    // "citizen ID number"
   if (firstIndex !== -1 && startIndex !== -1 && endIndex !== -1) {
     let textStr = data.value.slice(0, startIndex) + data.value.slice(endIndex);
     // ...
   }
   ```

   The three markers double as a "this really is an ID card" test: until all
   three are present, no extraction is attempted, which is what stops a random
   frame from producing a result.

10. **Extract and validate the entities.** `getEntity` proposes; the regular
    expressions dispose:

    ```ts
    let nameRegexp = new RegExp('^[\u4e00-\u9fa5]{2,4}$');
    let idNumberRegexp =
      new RegExp('^[1-9]\\d{5}(18|19|20)\\d{2}(0[1-9]|1[0-2])(0[1-9]|[12][0-9]|3[01])\\d{3}[\\dXx]$');

    textProcessing.getEntity(textStr, { entityTypes: [EntityType.NAME, EntityType.ID_NO] })
      .then((entities: textProcessing.Entity[]) => {
        let model = new IdCardModel();
        entities.forEach((entity: textProcessing.Entity) => {
          if (entity.type === EntityType.NAME && nameRegexp.test(entity.text)) {
            model.idName = entity.text;
          } else if (entity.type === EntityType.ID_NO && idNumberRegexp.test(entity.text)) {
            model.idNumber = entity.text;
          }
        });
        if (model.idName && model.idNumber && !GlobalContext.get().getT<boolean>('hasScanResult')) {
          AppStorage.setOrCreate('idCardModel', model);
          GlobalContext.get().setObject('hasScanResult', true);
          this.imageReceiver?.off('imageArrival');
        }
      })
      .catch((err: BusinessError) => {
        Logger.error(TAG, `getEntity errorCode: ${err.code} errorMessage: ${err.message}`);
      });
    ```

    Both fields must validate before anything is published - a half-read card is
    discarded and the next frame is tried. Do not log `textStr`, `entities` or
    the model (HW-02-0163).

11. **Publish through `AppStorage` and let `@Watch` do the navigation.**

    ```ts
    // ScanPage
    @Watch('onScanResult') @StorageProp('idCardModel') idCardModel: IdCardModel | undefined = undefined;

    onScanResult() {
      if (this.idCardModel) {
        this.pageInfos.pop();
      }
    }
    ```

    `MainPage` holds the identical declaration and copies the two values into
    its own `@State` fields. Neither page calls the service to ask for a result.

12. **Tear down once, and completely.** Corrected for HW-02-0164 and
    HW-02-0162:

    ```ts
    // ScanPage: one lifecycle callback owns the release
    .onWillHide(async () => {
      isInit = false;
      await CameraService.releaseCamera();
    })
    ```

    ```ts
    // CameraService.releaseCamera(), after the four existing try/catch blocks
    try {
      this.imageReceiver?.off('imageArrival');
      await this.imageReceiver?.release();
    } catch (error) {
      let err = error as BusinessError;
      Logger.error(TAG, `imageReceiver release fail: ${JSON.stringify(err)}`);
    } finally {
      this.imageReceiver = undefined;
    }
    this.offCameraStatusChange();
    ```

    Order matters: outputs and session first, then the input, then the receiver,
    then the camera-status subscription.

## Verified snippets

All snippets below are copied from the ZIP, not from the document.

`ScanIdCard.zip#entry/src/main/ets/pages/ScanPage.ets:27-43` - the controller
subclass that starts the camera when the surface exists:

```ts
class MyXComponentController extends XComponentController {
  onSurfaceCreated(surfaceId: string): void {
    Logger.info(TAG, `onSurfaceCreated surfaceId: ${surfaceId}`);
    GlobalContext.get().setObject('xComponentSurfaceId', surfaceId);
    isInit = true;
    CameraService.initCamera(surfaceId);
  }

  onSurfaceChanged(surfaceId: string, rect: SurfaceRect): void {
    Logger.info(TAG, `onSurfaceChanged surfaceId: ${surfaceId}, rect: ${JSON.stringify(rect)}}`);
  }

  onSurfaceDestroyed(surfaceId: string): void {
    Logger.info(TAG, `onSurfaceDestroyed surfaceId: ${surfaceId}`);
    GlobalContext.get().setObject('xComponentSurfaceId', '');
  }
}
```

`ScanIdCard.zip#entry/src/main/ets/service/CameraService.ets:139-152` - the
second channel: an `ImageReceiver` turned into a preview output:

```ts
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
      // Monitor preview events.
      this.imageReceiverCallBack(this.imageReceiver);
      // Monitor preview events.
      this.previewOutputCallBack(this.imageReceiverOutput);
```

`ScanIdCard.zip#entry/src/main/ets/service/CameraService.ets:306-323` - both
outputs added to a single session:

```ts
      if (this.curSceneMode === camera.SceneMode.NORMAL_PHOTO) {
        this.session = cameraManager.createSession(this.curSceneMode) as camera.PhotoSession;
      } else if (this.curSceneMode === camera.SceneMode.NORMAL_VIDEO) {
        this.session = cameraManager.createSession(this.curSceneMode) as camera.VideoSession;
      }
      if (this.session === undefined) {
        return;
      }
      this.onSessionErrorChange(this.session);
      this.session.beginConfig();
      this.session.addInput(cameraInput);
      this.session.addOutput(previewOutput);
      this.session.addOutput(imageReceiverOutput);
      await this.session.commitConfig();
      this.setFocusMode(camera.FocusMode.FOCUS_MODE_CONTINUOUS_AUTO);
      await this.session.start();
```

`ScanIdCard.zip#entry/src/main/ets/service/CameraService.ets:400-405` - the
three-marker layout filter that decides a frame really shows an ID card:

```ts
            let firstIndex = data.value.indexOf('姓名');
            let startIndex = data.value.indexOf('性别');
            let endIndex = data.value.indexOf('\n公民身份号码');
            if (!GlobalContext.get().getT<boolean>('hasScanResult') && firstIndex !== -1 && startIndex !== -1 &&
              endIndex !== -1) {
              let textStr = data.value.slice(0, startIndex) + data.value.slice(endIndex);
```

`ScanIdCard.zip#entry/src/main/ets/service/CameraService.ets:30-33` - the two
validators that gate the extracted entities:

```ts
// 定义姓名的正则表达式
let nameRegexp = new RegExp('^[\u4e00-\u9fa5]{2,4}$');
// 定义身份证号的正则表达式
let idNumberRegexp = new RegExp('^[1-9]\\d{5}(18|19|20)\\d{2}(0[1-9]|1[0-2])(0[1-9]|[12][0-9]|3[01])\\d{3}[\\dXx]$');
```

`ScanIdCard.zip#entry/src/main/ets/service/CameraService.ets:413-427` - publish
once, then stop the pipeline at the source:

```ts
                  entities.forEach((entity: textProcessing.Entity) => {
                    if (entity.type === EntityType.NAME && nameRegexp.test(entity.text)) {
                      // 姓名
                      model.idName = entity.text;
                    } else if (entity.type === EntityType.ID_NO && idNumberRegexp.test(entity.text)) {
                      // 身份证号码
                      model.idNumber = entity.text;
                    }
                  });
                  if (model.idName && model.idNumber && !GlobalContext.get().getT<boolean>('hasScanResult')) {
                    Logger.info(TAG, `Succeeded in get idCard msg：${JSON.stringify(model)}`);
                    AppStorage.setOrCreate('idCardModel', model);
                    GlobalContext.get().setObject('hasScanResult', true);
                    this.imageReceiver?.off('imageArrival');
                  }
```

The `Logger.info` line in that block is HW-02-0163 - delete it when you copy
this.

`ScanIdCard.zip#entry/src/main/ets/pages/MainPage.ets:96-116` - permission then
navigation, with the result state cleared on the way in:

```ts
          .onClick(() => {
            Logger.info(TAG, 'Enter start Camera.');
            this.clickFlag = true;
            let atManager = abilityAccessCtrl.createAtManager();
            atManager.requestPermissionsFromUser(this.getUIContext().getHostContext(), [
              'ohos.permission.CAMERA'
            ]).then((result: PermissionRequestResult): void => {
              if (result.authResults[0] === 0) {
                Logger.info(TAG, 'request Permissions success!');
                GlobalContext.get().setObject('hasScanResult', false);
                AppStorage.setOrCreate('idCardModel', undefined);
                this.pathStack.pushPathByName('ScanPage', null);
              } else {
                this.getUIContext().getPromptAction().showToast({ message: '请先授予相机权限。' });
              }
            }).catch((error: BusinessError): void => {
              Logger.info(TAG, `requestPermissionsFromUser call Failed! error: ${error.code}`);
            }).finally(() => {
              this.clickFlag = false;
            });
          });
```

`ScanIdCard.zip#entry/src/main/ets/pages/ScanPage.ets:47-56` - the result
listener that pops the page:

```ts
  @Watch('onScanResult') @StorageProp('idCardModel') idCardModel: IdCardModel | undefined = undefined;
  pageInfos: NavPathStack = new NavPathStack();
  private mXComponentController: XComponentController = new MyXComponentController();

  onScanResult() {
    Logger.info(TAG, 'Enter ScanPage onScanResult.');
    if (this.idCardModel) {
      this.pageInfos.pop();
    }
  }
```

## Permissions & config

One permission, user-grant, requested at runtime:

```json5
// entry/src/main/module.json5
"requestPermissions": [
  {
    "name": "ohos.permission.CAMERA",
    "reason": "$string:reason_camera",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
  }
]
```

Nothing else is needed. There is no AppGallery Connect configuration, no client
id, no metadata block, and `oh-package.json5` has an empty `dependencies` at
both the project and the module level - Camera Kit, Core Vision Kit and Natural
Language Kit are all SDK kits.

Also required in `module.json5`:

```json5
"pages": "$profile:main_pages",
"routerMap": "$profile:router_map",
```

The build targets API 20:

```json5
// build-profile.json5
"targetSdkVersion": "6.0.0(20)",
"compatibleSdkVersion": "6.0.0(20)",
```

## Constraints

- **API 20 / HarmonyOS 6.0.0 Release and later**, DevEco Studio 6.0.0 Release
  and later, phone only (`"deviceTypes": ["phone"]`).
- **Physical device only.** The entity extraction guide states plainly: "This
  capability is currently not supported on the Emulator." The custom camera
  preview is device-only as well. The document's constraints section says
  neither (HW-02-0169).
- **Entity extraction takes at most 1,000 characters** and supports simplified
  Chinese, English and traditional Chinese. An ID card is far below the limit,
  but the same pipeline pointed at a dense document is not.
- **The recognisers are Chinese-ID specific.** The three layout markers
  (`姓名` / `性别` / `公民身份号码`) and both regular expressions - two to four
  Han characters for the name, the eighteen-digit checksum-less pattern for the
  number - only match a mainland Chinese ID card. Any other document falls
  through silently.
- **The requested preview profile is a preference.** `getPreviewProfile` falls
  back to `previewProfiles[0]` when 1920x1080 in format 1003 is not offered, so
  the actual size must be read from `this.previewProfileObj` after
  `initCamera` assigns it.
- **The ImageReceiver holds eight buffers.** Every `nextImage` you take out must
  be released or the ninth frame never arrives.
- **`createImageReceiver`'s size and format do not configure anything.** The
  guide states the parameters "do not have a practical impact"; the pixel format
  and resolution come from the preview profile on the producer side.
- **The frame buffer may be row-padded.** `imgComponent.rowStride` is the
  authority, not the profile width.

## Pitfalls

1. **HW-02-0158 - the sample builds the preview with the deprecated XComponent
   constructor.** `ScanPage.ets:66-70` uses
   `XComponent({ id: 'componentId', type: XComponentType.SURFACE, controller })`.
   The component reference marks that overload "deprecated since API version 12.
   You are advised to use `XComponent(options: XComponentOptions)` instead", and
   this project targets API 20. Use
   `XComponent({ type: XComponentType.SURFACE, controller: this.mXComponentController })`
   instead; nothing in the sample reads the component id.

2. **HW-02-0168 - the document says to get the surface id with
   `getXComponentSurfaceId()`, which is not what the sample does and not what
   the reference advises.** Step 1 of the document shows
   `this.surfaceId = this.mXComponentController.getXComponentSurfaceId();`
   followed by `await CameraService.initCamera(this.surfaceId)`. No `surfaceId`
   field and no such call exist in the ZIP - `ScanPage.ets:28-33` takes the id
   from `onSurfaceCreated`. The reference warns that `onLoad` fires before
   `onSurfaceCreated`, so the id may not be valid there. Follow the ZIP.

3. **HW-02-0160 - the row stride is never checked, which is incorrect on any
   device that pads preview rows.** `CameraService.ets:372-385` hands
   `imgComponent.byteBuffer` straight to `image.createImageSource` with
   `sourceSize` taken from the profile. The ImageReceiver guide reads
   `imgComponent.rowStride` in exactly this callback and says: "Check whether
   the image width matches the row stride. If they do not match, you can
   preprocess the data using either of the two methods outlined below."
   Where stride differs from width, repack row by row before decoding, or decode
   at `width = stride` and `cropSync` the result. Skipping the check shears
   every frame and the OCR then never matches the card.

4. **HW-02-0161 - two early returns skip `nextImage.release()`, and eight of
   them stop the scanner dead.** `CameraService.ets:440` is the only release,
   and `:368-371` (getComponent error) and `:436-439` (`byteBuffer` not an
   `ArrayBuffer`) both `return` before reaching it. The receiver has eight
   buffers; once eight frames leak, `imageArrival` stops firing while the
   preview keeps running, so the failure looks like "recognition just stopped".
   Release on every path.

5. **HW-02-0159 - a full-resolution PixelMap is decoded per frame and never
   released.** `CameraService.ets:385` creates it; `pixelMap.release()` appears
   nowhere in the ZIP. The image decoding guide says: "If your application
   handles the PixelMap instance on its own, you are advised to manually release
   the PixelMap instance." Because the bitmap is never given to an `Image`
   component, nothing else can free it either. Release it in a `.finally()` on
   the recognition chain.

6. **HW-02-0165 - there is no in-flight guard, so recognitions stack up.** The
   only test before starting a recognition is
   `if (!GlobalContext.get().getT<boolean>('hasScanResult'))`
   (`CameraService.ets:359-361`), and `hasScanResult` is not set until a full
   recognition **plus** entity extraction has already succeeded. Frames arrive
   faster than `recognizeText` resolves, so each one launches another
   recognition holding another undisposed bitmap. Add an `isRecognizing` flag
   and clear it in the same `finally` that releases the PixelMap.

7. **HW-02-0162 - `releaseCamera` forgets the ImageReceiver entirely.**
   `CameraService.ets:181-217` releases the preview output, the receiver output,
   the session and the camera input, and unsubscribes `cameraStatus`. It never
   calls `this.imageReceiver.release()` and never calls
   `off('imageArrival')`, and it never sets `this.imageReceiver = undefined`.
   `initCamera` calls `releaseCamera` and then builds a **new** receiver at
   `:140` and subscribes again at `:150`, so backing out of the scan page
   without a result - the ordinary failure - leaks a receiver with eight
   full-resolution buffers plus a live listener, once per attempt. The sample
   proves `off` exists by calling it on the success path at `:426`.

8. **HW-02-0164 - the teardown is wired to two lifecycle callbacks at once.**
   `ScanPage.ets:119-123` (`onHidden`) and `:124-127` (`onWillHide`) both
   `await CameraService.releaseCamera()`. Neither awaits the other and
   `releaseCamera` only clears its handles in its `finally` blocks, so the
   second call reads handles that are already being released. The errors are
   swallowed by the surrounding catches, so the double teardown never shows up
   in the log. Bind the release to `onWillHide` alone and leave `onHidden` to
   reset `isInit`.

9. **HW-02-0163 - the ID card content is written to the log as public
   plaintext.** `CameraService.ets:406` logs the whole recognised card text,
   `:411` logs the extracted entities and `:423` logs
   `JSON.stringify(model)` - the person's name and national ID number. All of
   them pass through `Logger.ets:24`, whose format string is
   `'%{public}s, %{public}s'`, and the hilog reference states: "Parameters
   labeled {public} are public data and are displayed in plaintext; parameters
   labeled {private} (default value) are private data and are filtered by
   `<private>`." In a feature whose entire subject is an identity document this
   is the wrong default. Delete the three payload logs.

10. **HW-02-0166 - the logger declares two format identifiers and supplies one
    array.** `Logger.ets:24` sets `'%{public}s, %{public}s'` while every method
    calls `hilog.info(this.domain, this.prefix, this.format, args)` with `args`
    as a single value. The hilog reference requires that "the number and type of
    parameters must map to the identifier in the format string." Spread it:
    `hilog.info(this.domain, this.prefix, this.format, ...args)`.

11. **HW-02-0169 - the constraints section omits the emulator restriction.**
    All three bullets under 约束与限制 are version statements. The entity
    extraction guide the document links to opens its own constraints with "This
    capability is currently not supported on the Emulator", and the custom
    camera preview needs a physical device too. On the emulator the preview runs
    and nothing is ever recognised, with no explanation anywhere in the
    document.

12. **HW-02-0167 - four user-visible strings are hardcoded while the rest come
    from `string.json`.** `MainPage.ets:109`, `:227`, `:235` and
    `ScanPage.ets:89-91` embed Chinese literals for the permission-denied toast,
    the confirm button, the submit toast and the scanner instruction, while ten
    other strings in the same two files use `$r('app.string.*')`. Move them into
    the resource file.

13. **HW-02-0170 - the two convenient-life OCR samples disagree on the
    `textRecognition` lifecycle.** This sample calls `recognizeText`
    (`CameraService.ets:395`) with no `init` and no `release` anywhere in the
    ZIP. `LIFE-21`'s sample brackets it with
    `await textRecognition.init()` in `aboutToAppear` and
    `await textRecognition.release()` in `aboutToDisappear`
    (`CourierAddressIdentification.zip#entry/src/main/ets/pages/Index.ets:59` and
    `:65`). Both documents describe the same 通用文字识别 capability and link to
    the same guide, so only one can be the intended pattern. Until the platform
    documentation says which, treat the lifecycle as unsettled - do not assume
    the per-frame call site here is safe to leave un-initialised just because
    this sample does.

## References

- Document:
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/id_card_scan-0000002338766518
- XComponent (constructor overloads, `getXComponentSurfaceId`, `onSurfaceCreated`):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-xcomponent
- Using ImageReceiver to Receive Images (stride check, `nextImage.release`):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-receiver
- Image Decoding (when to release ImageSource and PixelMap):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-decoding
- Camera Kit overview:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/camera-overview
- Core Vision Kit / general text recognition:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/core-vision-text-recognition
- Natural Language Kit / entity extraction (emulator constraint, 1,000-character
  limit, entity types):
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/natural-language-getentity
- NavDestination lifecycle (`onWillHide`, `onHidden`):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navdestination
- hilog (`{public}` vs `{private}`, argument mapping):
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
- ohos.permission.CAMERA:
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/permissions-for-all-user#ohospermissioncamera
