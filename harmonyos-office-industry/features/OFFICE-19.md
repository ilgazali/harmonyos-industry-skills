---
id: OFFICE-19
title: Watermark camera - custom capture, GPS EXIF read-back, OffscreenCanvas watermark and SaveButton save
industry: 05_office
doc: huawei_industry_tree/05_office/docs/19_watermark_camera.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/watermark_camera-0000002334789536
sample: huawei_industry_tree/05_office/downloads/WatermarkCamera.zip
kits: ["@kit.CameraKit", "@kit.LocationKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.CoreFileKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [XComponent, XComponentType, XComponentController, ImageAIOptions, ImageAnalyzerType, "camera.getCameraManager", "CameraManager.getSupportedCameras", "CameraManager.getSupportedSceneModes", "CameraManager.getSupportedOutputCapability", "CameraManager.createCameraInput", "CameraManager.createPreviewOutput", "CameraManager.createPhotoOutput", "CameraManager.createSession", "camera.SceneMode.NORMAL_PHOTO", "PhotoSession.beginConfig", "PhotoSession.addInput", "PhotoSession.addOutput", "PhotoSession.commitConfig", "PhotoSession.start", "PhotoSession.hasFlash", "PhotoSession.setFlashMode", "PhotoSession.isFocusModeSupported", "PhotoSession.setFocusMode", "PhotoOutput.capture", "PhotoOutput.on('photoAssetAvailable')", "camera.PhotoCaptureSetting", "camera.Location", "camera.Profile", "geoLocationManager.getCurrentLocation", "geoLocationManager.LocationRequest", "geoLocationManager.LocationRequestPriority", "image.createImageSource", "ImageSource.getImageProperty", "image.PropertyKey.GPS_LATITUDE", "image.PropertyKey.GPS_LONGITUDE", "ImageSource.getImageInfo", "ImageSource.createPixelMap", "ImageSource.release", "PixelMap.release", OffscreenCanvas, "OffscreenCanvasRenderingContext2D.drawImage", "OffscreenCanvasRenderingContext2D.fillText", "OffscreenCanvasRenderingContext2D.getPixelMap", SaveButton, SaveDescription, SaveButtonOnClickResult, "photoAccessHelper.getPhotoAccessHelper", "PhotoAccessHelper.createAsset", "image.createImagePacker", "ImagePacker.packToData", "fileIo.open", "fileIo.truncate", "fileIo.write", "fileIo.close", "display.getDefaultDisplaySync"]
permissions: ["ohos.permission.CAMERA", "ohos.permission.LOCATION", "ohos.permission.APPROXIMATELY_LOCATION", "ohos.permission.MEDIA_LOCATION"]
min_api: 20
modules: [entry]
findings: [HW-05-0103, HW-05-0104, HW-05-0105, HW-05-0106, HW-05-0107, HW-05-0108, HW-05-0109, HW-05-0110, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when a photo has to carry **provable location evidence** - an
attendance check-in shot, a site inspection, a delivery proof - so the position
must be both **written into the file's EXIF** and **burned into the visible
pixels**.

The chain is longer than it looks, and the ordering is the interesting part:

1. Custom capture on an `XComponent` surface (not the system camera picker,
   because the app needs control of the capture settings).
2. Get a location fix and put it in `PhotoCaptureSetting.location` **before**
   the shutter, so Camera Kit writes it into the photo's EXIF.
3. When the asset arrives, **read the coordinates back out of the file** with
   `ImageSource.getImageProperty(GPS_LATITUDE / GPS_LONGITUDE)` rather than
   reusing the in-memory values - that proves what actually landed in the file.
4. Draw the photo plus the coordinate text onto an `OffscreenCanvas` and take a
   new `PixelMap` from it.
5. Save with a `SaveButton` security component, which grants the one-shot media
   write without a media permission.

## Feature checklist

The application must be able to:

- Preview the camera on an `XComponent` surface and switch front/rear.
- Request camera and location permissions before capture.
- Obtain a location fix and attach it to the capture settings so it reaches EXIF.
- Capture even when the location is unavailable, telling the user why the stamp
  is missing.
- Receive the captured asset, open it, and read `GPS_LATITUDE` / `GPS_LONGITUDE`
  back from the file.
- Decode the photo to a `PixelMap`, draw it and the coordinate text onto an
  `OffscreenCanvas`, and produce a watermarked `PixelMap`.
- Show the result in a preview dialog with a `SaveButton`.
- Save the watermarked image into the media library.
- Release the camera pipeline, the file descriptors and the image objects.

## Architecture

Single `entry` HAP:

| File | Responsibility |
| --- | --- |
| `pages/CameraPage.ets` | the camera screen: `XComponent`, shutter, lens switch, dialog host |
| `utils/CameraUtils.ets` | **the whole feature** - `cameraShooting`, `capture`, `releaseCamera`, `setPhotoOutputCb`, `pixelMap`, `imageSource2PixelMap`, `addWatermark`, and the exported `saveToFile` |
| `component/DialogBuilder.ets` | the preview dialog with the `SaveButton` |
| `component/FlashBlackComponent.ets` | the shutter flash overlay |
| `constants/Constants.ets` | profile sizes/formats and the watermark label strings |
| `entryability/`, `entrybackupability/` | ability entry and backup stub |

Note the directory is `component/` (singular) - the document's tree says
`components/` (HW-05-0110).

`CameraUtils` is a singleton class plus two module-level variables that carry
state between stages:

```ts
let fd: number | null = null;                                  // saveToFile's descriptor
let addedWatermarkPixelMap: image.PixelMap | null = null;      // the watermark result
```

Both are the source of defects (HW-05-0105, HW-05-0109) - they are the pattern to
avoid, not to copy.

Capture-to-save flow:

```
shutter -> capture(isFront)
  geoLocationManager.getCurrentLocation(requestInfo, (err, location) => {
    PhotoCaptureSetting { location: {lat, lon, alt}, quality: HIGH, rotation: 0, mirror: isFront }
    photoOutPut.capture(settings)
  })

photoOutput.on('photoAssetAvailable') fires
  -> pixelMap(photoAsset.uri)
       fs.openSync(uri, READ_ONLY)
       image.createImageSource(fd)
       getImageProperty(GPS_LATITUDE) / getImageProperty(GPS_LONGITUDE)
       imageSource2PixelMap -> getImageInfo + createPixelMap
       addWatermark(pixelMap, 'lat...\nlon...\n', uiContext)
            OffscreenCanvas(width, height)
            drawImage(pixelMap) + fillText(text)
            addedWatermarkPixelMap = ctx.getPixelMap(...)
       AppStorage.setOrCreate('locationUri', addedWatermarkPixelMap)
  -> AppStorage.set('isDialogOpen', true)

dialog SaveButton onClick (result === SUCCESS)
  -> saveToFile(image, context)
       phAccessHelper.createAsset(IMAGE, 'png')
       imagePacker.packToData(pixelMap, { format: 'image/png', quality: 100, needsPackProperties: true })
       fileIo.open -> truncate -> write -> close
```

## Implementation steps

1. **Declare the four permissions.** `CAMERA` for the custom pipeline,
   `LOCATION` **and** `APPROXIMATELY_LOCATION` together (the precise permission
   cannot be requested alone), and `MEDIA_LOCATION` for reading the geolocation
   out of the saved media file.
2. **Preview on an `XComponent`.** `XComponentType.SURFACE` with a controller;
   take the surfaceId in `onLoad` and pass it to `createPreviewOutput`.
3. **Select profiles that the device actually supports.** Search
   `cameraOutputCap.previewProfiles` / `photoProfiles`, and **check the search
   result before passing it** to `createPreviewOutput` / `createPhotoOutput` - the
   sample's hard-coded 1440x1080 constants are, by its own comment, device
   dependent (HW-05-0104).
4. **Register the asset callback before starting**, and unregister it in the
   teardown (HW-05-0107).
5. **Assemble and start the session**: `beginConfig` -> `addInput` ->
   `addOutput` (preview) -> `addOutput` (photo) -> `commitConfig` -> `start`,
   then probe `hasFlash` and `isFocusModeSupported` before setting either.
6. **Get the fix, then shoot.** `getCurrentLocation` with a
   `LocationRequestPriority.FIRST_FIX` request; put the coordinates in
   `PhotoCaptureSetting.location`. Do **not** cancel the capture when the fix
   fails - report it and shoot without the stamp (HW-05-0108).
7. **Read the coordinates back from the file.** Open the asset URI `READ_ONLY`,
   `image.createImageSource(fd)`, then `getImageProperty(PropertyKey.GPS_LATITUDE)`
   and `GPS_LONGITUDE` with a `defaultValue`.
8. **Decode and draw.** `getImageInfo` for the size, `createPixelMap` with
   `editable: true`, then an `OffscreenCanvas` sized in **vp** (`px2vp` the pixel
   dimensions), `drawImage`, `fillText`, and `getPixelMap` for the result. Scale
   the text padding by `imageWidth / displayWidthInVp` so the stamp keeps its
   relative size across resolutions.
9. **Return the watermark rather than parking it in a module variable**, and
   await the call (HW-05-0109).
10. **Release everything.** The camera pipeline with awaited releases
    (HW-05-0103), the `ImageSource` and both `PixelMap`s (HW-05-0106), and the
    save descriptor exactly once from a local variable (HW-05-0105).
11. **Save through `SaveButton`.** The security component grants the write when
    `result === SaveButtonOnClickResult.SUCCESS`; `createAsset` + `packToData` +
    `fileIo.write` does the rest.

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Session assembly with capability-based profile selection

`WatermarkCamera.zip#entry/src/main/ets/utils/CameraUtils.ets`

```ts
async cameraShooting(cameraPosition: number, surfaceId: string, context: UIContext): Promise<void> {
  this.releaseCamera();                                   // not awaited - HW-05-0103
  this.uiContext = context;
  let cameraManager: camera.CameraManager = camera.getCameraManager(context.getHostContext());
  if (!cameraManager) { return; }

  let cameraArray: camera.CameraDevice[] = cameraManager.getSupportedCameras();
  if (cameraArray.length <= 0) { return; }

  this.cameraInput = cameraManager.createCameraInput(cameraArray[cameraPosition]);
  await this.cameraInput.open();
  let sceneModes: camera.SceneMode[] = cameraManager.getSupportedSceneModes(cameraArray[cameraPosition]);
  let cameraOutputCap: camera.CameraOutputCapability =
    cameraManager.getSupportedOutputCapability(cameraArray[cameraPosition], camera.SceneMode.NORMAL_PHOTO);
  if (!cameraOutputCap) { return; }

  let isSupportPhotoMode: boolean = sceneModes.indexOf(camera.SceneMode.NORMAL_PHOTO) >= 0;
  if (!isSupportPhotoMode) { return; }

  let previewProfile: undefined | camera.Profile = cameraOutputCap.previewProfiles.find((profile: camera.Profile) => {
    return profile.size.width === this.previewProfileObj.size.width &&
      profile.size.height === this.previewProfileObj.size.height &&
      profile.format === this.previewProfileObj.format;
  });
  // ... photoProfile searched the same way

  this.previewOutput = cameraManager.createPreviewOutput(previewProfile, surfaceId);   // HW-05-0104
  if (this.previewOutput === undefined) { return; }
  this.photoOutPut = cameraManager.createPhotoOutput(photoProfile);                    // HW-05-0104
  if (this.photoOutPut === undefined) { return; }

  // 注册监听photoAsset上报
  await this.setPhotoOutputCb(this.photoOutPut);

  this.photoSession = cameraManager.createSession(camera.SceneMode.NORMAL_PHOTO) as camera.PhotoSession;
  if (this.photoSession === undefined) { return; }
  this.photoSession.beginConfig();
  this.photoSession.addInput(this.cameraInput);
  this.photoSession.addOutput(this.previewOutput);
  this.photoSession.addOutput(this.photoOutPut);

  await this.photoSession.commitConfig();
  await this.photoSession.start();
  let flashStatus: boolean = this.photoSession.hasFlash();
  if (flashStatus) {
    this.photoSession.setFlashMode(camera.FlashMode.FLASH_MODE_CLOSE);
  }
  let focusModeStatus: boolean = this.photoSession.isFocusModeSupported(camera.FocusMode.FOCUS_MODE_CONTINUOUS_AUTO);
  if (focusModeStatus) {
    this.photoSession.setFocusMode(camera.FocusMode.FOCUS_MODE_CONTINUOUS_AUTO);
  }
}
```

Searching `cameraOutputCap` for a profile is the right idea and better than
OFFICE-18's hand-built `camera.Profile`; the missing piece is a check on the
search result. The searched constants are:

```ts
static readonly PHOTO_PROFILE_SIZE_WIDTH: number = 1440;
static readonly PHOTO_PROFILE_SIZE_HEIGHT: number = 1080;
static readonly PHOTO_PROFILE_FORMAT: number = 2000;
static readonly PREVIEW_PROFILE_SIZE_WIDTH: number = 1440;
static readonly PREVIEW_PROFILE_SIZE_HEIGHT: number = 1080;
static readonly PREVIEW_PROFILE_FORMAT: number = 1003;
```

### Location into the capture settings

`WatermarkCamera.zip#entry/src/main/ets/utils/CameraUtils.ets`

```ts
capture(isFront: boolean): void {
  let requestInfo: geoLocationManager.LocationRequest = {
    priority: geoLocationManager.LocationRequestPriority.FIRST_FIX,
    scenario: geoLocationManager.LocationRequestScenario.UNSET
  };
  try {
    geoLocationManager.getCurrentLocation(requestInfo, async (err, location) => {
      if (err) {
        Logger.info(TAG, err.message + err.code);
        return;                                    // cancels the capture - HW-05-0108
      }
      let captureLocation: camera.Location = {
        latitude: location.latitude,
        longitude: location.longitude,
        altitude: location.altitude
      };
      // 保存坐标用于后续拍照
      let settings: camera.PhotoCaptureSetting = {
        location: captureLocation,
        quality: camera.QualityLevel.QUALITY_LEVEL_HIGH,
        rotation: camera.ImageRotation.ROTATION_0,
        mirror: isFront
      };
      if (this.photoOutPut) {
        this.photoOutPut.capture(settings);
      }
    });
  } catch (error) {
    Logger.error('failed to get location:', error);
  }
}
```

`PhotoCaptureSetting.location` is the mechanism that gets the coordinates into
the file's EXIF - that is why the fix is taken *before* the shutter rather than
stamped afterwards.

### Reading the coordinates back out of the file

`WatermarkCamera.zip#entry/src/main/ets/utils/CameraUtils.ets`

```ts
async setPhotoOutputCb(photoOutput: camera.PhotoOutput): Promise<void> {
  photoOutput.on('photoAssetAvailable',                       // never off - HW-05-0107
    async (_err: BusinessError, photoAsset: photoAccessHelper.PhotoAsset): Promise<void> => {
      this.uri = photoAsset.uri;
      await this.pixelMap(this.uri);
      AppStorage.set<boolean>('isDialogOpen', true);
    });
}

// 根据uris创建imageSource，获取ImageProperty属性
async pixelMap(photoUri: string) {
  let file: fileIo.File | undefined = undefined;
  try {
    file = fs.openSync(photoUri, fs.OpenMode.READ_ONLY);
    let imageSource: image.ImageSource = image.createImageSource(file.fd);
    let options: image.ImagePropertyOptions = { index: 0, defaultValue: '0' };
    let latitude = await imageSource.getImageProperty(image.PropertyKey.GPS_LATITUDE, options);
    let longitude = await imageSource.getImageProperty(image.PropertyKey.GPS_LONGITUDE, options);
    let cameraImage: ImagePixelMap = await this.imageSource2PixelMap(imageSource);
    if (this.uiContext) {
      this.addWatermark(cameraImage, Constants.LATITUDE + latitude + '\n' + Constants.LONGITUDE + longitude + '\n',
        this.uiContext);                                       // not awaited - HW-05-0109
    }
    AppStorage.setOrCreate('locationUri', addedWatermarkPixelMap);
  } finally {
    if (file) {
      fs.closeSync(file);
    }
  }
}
```

The `defaultValue: '0'` in `ImagePropertyOptions` is what keeps the read from
throwing when the photo carries no GPS tag - a photo taken without a fix stamps
`0` rather than failing. The `try/finally` around the descriptor here is correct;
`saveToFile` is the one that gets it wrong.

### Decode and watermark on an OffscreenCanvas

`WatermarkCamera.zip#entry/src/main/ets/utils/CameraUtils.ets`

```ts
async imageSource2PixelMap(imageSource: image.ImageSource): Promise<ImagePixelMap> {
  let imageInfo: image.ImageInfo = await imageSource.getImageInfo();
  let height = imageInfo.size.height;
  let width = imageInfo.size.width;
  let options: image.DecodingOptions = {
    editable: true,
    desiredSize: { height, width }
  };
  let pixelMap: PixelMap = await imageSource.createPixelMap(options);
  let result: ImagePixelMap = { pixelMap, width, height };
  return result;
}

async addWatermark(imagePixelMap: ImagePixelMap, text: string, uiContext: UIContext) {
  let height = uiContext.px2vp(imagePixelMap.height) as number;
  let width = uiContext.px2vp(imagePixelMap.width) as number;
  let offScreenCanvas = new OffscreenCanvas(width, height);
  let offScreenContext = offScreenCanvas.getContext('2d');
  offScreenContext.drawImage(imagePixelMap.pixelMap, 0, 0, width, height);
  let displayWidth = display.getDefaultDisplaySync().width;
  let vpWidth = uiContext.px2vp(displayWidth);
  let imageScale = width / vpWidth;
  offScreenContext.textAlign = 'left';
  offScreenContext.fillStyle = '#A2FFFFFF';
  offScreenContext.font = '50px';
  let padding = 5 * imageScale;
  offScreenContext.fillText(text, 3 * padding, 5 * padding);
  addedWatermarkPixelMap = offScreenContext.getPixelMap(0, 0, width, height);
}
```

Two details worth keeping: `editable: true` in the decoding options (an
uneditable `PixelMap` cannot be drawn into a canvas destination), and the
`imageScale = imageWidthInVp / displayWidthInVp` factor that keeps the stamp's
padding proportional to the photo rather than to the screen.

Corrected signature so the caller can await a returned value:

```ts
async addWatermark(imagePixelMap: ImagePixelMap, text: string, uiContext: UIContext): Promise<image.PixelMap> {
  // ... as above ...
  return offScreenContext.getPixelMap(0, 0, width, height);
}

// caller
const watermarked = await this.addWatermark(cameraImage, text, this.uiContext);
imagePixelMap.pixelMap.release();
imageSource.release();
AppStorage.setOrCreate('locationUri', watermarked);
```

### Saving through the SaveButton security component

`WatermarkCamera.zip#entry/src/main/ets/component/DialogBuilder.ets`

```ts
SaveButton({ text: SaveDescription.SAVE })
  .fontSize(18)
  .fontWeight(400)
  .height(40)
  .fontColor('rgb(10, 89, 247)')
  .backgroundColor($r('sys.color.comp_background_list_card'))
  .onClick(async (_event: ClickEvent, result: SaveButtonOnClickResult) => {
    if (result === SaveButtonOnClickResult.SUCCESS) {
      try {
        await saveToFile(dialog.image, dialog.context);
        dialog.save();
      } catch (err) {
        Logger.error('0x0000', 'createAsset failed, error:' + err);
      }
    } else {
      Logger.error('0x0000', 'SaveButtonOnClickResult createAsset failed');
    }
  });
```

`WatermarkCamera.zip#entry/src/main/ets/utils/CameraUtils.ets`

```ts
export async function saveToFile(pixelMap: image.PixelMap, context: Context): Promise<void> {
  try {
    let phAccessHelper = photoAccessHelper.getPhotoAccessHelper(context);
    let filePath = await phAccessHelper.createAsset(photoAccessHelper.PhotoType.IMAGE, 'png');
    let imagePacker = image.createImagePacker();
    let imageBuffer = await imagePacker.packToData(pixelMap, {
      format: 'image/png',
      quality: 100,
      needsPackProperties: true,
    });
    let mode = fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE;
    fd = (await fileIo.open(filePath, mode)).fd;      // module-level fd - HW-05-0105
    await fileIo.truncate(fd);
    await fileIo.write(fd, imageBuffer);
  } catch (err) {
    Logger.error(TAG, 'saveToFile error：', JSON.stringify(err) ?? '');
  } finally {
    if (fd) {                                         // skips fd 0, never reset
      fileIo.close(fd);                               // not awaited
    }
  }
}
```

`needsPackProperties: true` is what carries the EXIF - including the GPS tags -
through the re-pack into the saved PNG. Corrected descriptor handling:

```ts
let file: fileIo.File | undefined = undefined;
try {
  // ...
  file = await fileIo.open(filePath, mode);
  await fileIo.truncate(file.fd);
  await fileIo.write(file.fd, imageBuffer);
} catch (err) {
  Logger.error(TAG, 'saveToFile error: %{public}s', JSON.stringify(err));
} finally {
  if (file !== undefined) {
    fileIo.closeSync(file);
  }
}
```

## Permissions & config

`WatermarkCamera.zip#entry/src/main/module.json5`

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.CAMERA",
    "reason": "$string:reason_camera",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
  },
  {
    "name": "ohos.permission.LOCATION",
    "reason": "$string:reason_location",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
  },
  {
    "name": "ohos.permission.MEDIA_LOCATION",  // 访问媒体文件地理位置
    "reason": "$string:reason_add_location_to_media",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
  },
  {
    "name": "ohos.permission.APPROXIMATELY_LOCATION",
    "reason": "$string:reason_location",
    "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
  }
]
```

Notes on the config:

- The declared set matches the document's 权限说明 section exactly, and every
  entry has a distinct `reason` string - verified consistent.
- `ohos.permission.CAMERA` is genuinely required: this is a custom pipeline
  through `CameraManager`, not a system picker. Contrast OFFICE-05 (HW-05-0022)
  and OFFICE-16, where the same declaration was an over-declaration.
- `LOCATION` and `APPROXIMATELY_LOCATION` must be requested **together** - the
  precise permission cannot be granted on its own.
- `MEDIA_LOCATION` is what allows the geolocation to be read back out of the
  media file, which is exactly what `getImageProperty(GPS_LATITUDE)` does.
- **No media write permission** is declared, and none is needed: `SaveButton` is a
  security component that grants the one-shot write when the user presses it.
- Every entry uses `"when": "always"` although the camera and the location are
  only used while the capture page is in the foreground; `"inuse"` describes this
  scenario more accurately.
- `build-profile.json5` enables `caseSensitiveCheck: true`, which is why the
  `component/` vs `components/` discrepancy in the document matters
  (HW-05-0110).

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **The profile constants are device dependent.** The sample's own comment says
  so: `// 推荐的输出流配置，视机型而定`. 1440x1080 with formats 2000 / 1003 is a
  starting point, not a guarantee - always verify against
  `getSupportedOutputCapability`.
- **The location must be attached before the shutter.** `PhotoCaptureSetting.location`
  is what puts the coordinates into EXIF; there is no way to add them to the file
  afterwards through this API.
- **`getImageProperty` needs a `defaultValue`.** With `{ index: 0, defaultValue: '0' }`
  a photo with no GPS tag yields `'0'` instead of failing.
- **`createPixelMap` must be `editable: true`** for the result to be usable as a
  canvas source.
- **`needsPackProperties: true` preserves EXIF** through `packToData`; without it
  the saved PNG loses the GPS tags that the visible watermark claims.
- **`SaveButton` grants a single write.** The save only proceeds when the click
  result is `SaveButtonOnClickResult.SUCCESS`; the app cannot save without the
  user pressing the security component.
- **The camera is exclusive.** Front/rear switching re-enters `cameraShooting`,
  so the previous pipeline must be fully released first.
- **Image objects hold native memory.** `ImageSource` and `PixelMap` both have
  `release()`, and the image-decoding guide lists releasing them as a required
  step.

## Pitfalls

- **`releaseCamera` awaiting none of its five releases, and being called without
  `await`, is incorrect.** The new camera input is opened while the old one is
  still closing - on every lens switch. Await each release and await the call.
  (HW-05-0103)
- **Passing the result of `find()` straight into `createPreviewOutput` /
  `createPhotoOutput` is incorrect.** The variables are declared
  `undefined | camera.Profile` and the searched size/format is device dependent by
  the code's own comment; the `undefined` checks that follow examine the returned
  output, not the profile that was passed in. (HW-05-0104)
- **Keeping `saveToFile`'s descriptor in a module-level `fd` is incorrect** three
  ways: it is never reset, so a later failed `open` closes the previous
  descriptor; `if (fd)` skips a legitimate descriptor 0; and `fileIo.close` is not
  awaited. Use a local `fileIo.File` and an explicit `undefined` check.
  (HW-05-0105)
- **Never releasing the `ImageSource` or either `PixelMap` is incorrect.** The
  image-decoding guide makes releasing them step 5, and every shutter press here
  creates a full-resolution decode plus a full-resolution canvas readback.
  (HW-05-0106)
- **Registering `photoAssetAvailable` inside the pipeline setup with no `off` is
  incorrect** - each camera restart adds another handler, and that handler runs
  the whole decode-and-watermark chain. (HW-05-0107)
- **Returning early from the location callback is incorrect** as a shutter
  behaviour: `photoOutPut.capture` is only reached on the location success path,
  so with the location switch off the shutter silently does nothing. Capture
  anyway and tell the user the stamp is missing. (HW-05-0108)
- **Reading `addedWatermarkPixelMap` on the line after an un-awaited
  `addWatermark` is incorrect.** It works only because that method's body happens
  to contain no `await`; return the `PixelMap` and await the call instead.
  (HW-05-0109)
- **The project tree says `components/` while the sample ships `component/`,
  which is incorrect** - and the build has `caseSensitiveCheck: true`.
  (HW-05-0110)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-guides/02_media/image-decoding.md` - step 5, "Release
  the PixelMap and ImageSource instances", with the `pixelMap?.release();
  imageSource?.release();` helper and the note on when the `ImageSource` may be
  released.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-decoding
- `documentation/harmonyos-guides/02_media/camera-shooting.md` - the custom photo
  pipeline and the buffer-release requirement after capture.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/camera-shooting
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `OpenMode`, `open`, `truncate`, `write`, `close` / `closeSync`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-guides/04_application-services/map-location.md` -
  requesting `ohos.permission.LOCATION` together with
  `ohos.permission.APPROXIMATELY_LOCATION`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-location
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `usedScene`
  and the `inuse` / `always` values.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` -
  `SaveButton`, `SaveDescription` and `SaveButtonOnClickResult`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `documentation/harmonyos-references/04_media/arkts-apis-image-pixelmap.md` and
  `arkts-apis-image-imagesource.md` - the reference pages for the image objects
  used here; both are stubs in this repository, so the release contract above was
  taken from the image-decoding guide instead.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-pixelmap
