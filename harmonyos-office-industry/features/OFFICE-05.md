---
id: OFFICE-05
title: Personal card page - generate a QR code, snapshot it and save to the gallery via the authorization dialog
industry: 05_office
doc: huawei_industry_tree/05_office/docs/05_personal_card.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/personal_card-0000002236270234
sample: huawei_industry_tree/05_office/downloads/Personal_Card.zip
kits: ["@kit.ScanKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.CoreFileKit", "@kit.ArkUI", "@kit.ArkGraphics2D", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["generateBarcode.createBarcode", "generateBarcode.CreateOptions", "generateBarcode.ErrorCorrectionLevel", "scanBarcode.startScanForResult", "scanBarcode.ScanOptions", "scanCore.ScanType", "scanCore.ScanErrorCode", "UIContext.getComponentSnapshot", "ComponentSnapshot.get", "image.createImagePacker", "ImagePacker.packToData", "image.PackingOption", "fs.openSync", "fs.writeSync", "fs.closeSync", "fileUri.getUriFromPath", "photoAccessHelper.getPhotoAccessHelper", "PhotoAccessHelper.showAssetsCreationDialog", "photoAccessHelper.PhotoCreationConfig", "photoAccessHelper.PhotoType", "photoAccessHelper.PhotoSubtype", "fileIo.openSync", "fileIo.copyFileSync", "fileIo.closeSync", "uiEffect.createFilter", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.on('avoidAreaChange')", "AppStorage.setOrCreate", "@StorageProp", "UIContext.px2vp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-05-0022, HW-05-0023, HW-05-0024, HW-05-0025, HW-05-0026, HW-05-0027, HW-05-0028, HW-05-0029, HW-05-0184]
status: verified-with-fixes
---

## When to use

Load this card when an office app needs a **personal QR business card**: a page
that renders the user's own identity as a QR code, lets them save that code to
the system gallery, and lets them scan somebody else's code.

The interesting part is the save path. The app never writes to the gallery
directly and never asks for a media permission. Instead it:

1. snapshots the on-screen `Image` component into a `PixelMap`,
2. packs the `PixelMap` to PNG bytes and writes them to its own sandbox,
3. hands the sandbox URI to `showAssetsCreationDialog`, which asks the user and
   returns a **destination URI in the media library** that the app is
   temporarily allowed to write,
4. copies the sandbox file into that destination.

That is the permission-free way to save a user file, and it is the reason this
scenario declares no media permission at all.

## Feature checklist

The application must be able to:

- Generate a QR code from the card content in `aboutToAppear` and render it as a
  `PixelMap`.
- Capture the rendered QR code component by id into a `PixelMap` snapshot.
- Pack the snapshot to PNG and write it into the application sandbox.
- Raise the system save-authorization dialog and obtain a media-library
  destination URI.
- Copy the sandbox file into that destination and confirm with a toast.
- Report failure at every stage - packing, writing, dialog dismissal, copy - and
  release every file handle exactly once.
- Open the Scan Kit default UI to scan another person's QR code and surface both
  the result and any error.
- Lay out under the status bar and navigation indicator using the avoid-area
  heights.

## Architecture

Single `entry` HAP, five source files:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ts` | loads `pages/MainPage`, full-screen layout, publishes `topRectHeight` / `bottomRectHeight` into `AppStorage`, subscribes to `avoidAreaChange` |
| `pages/MainPage.ets` | `@Entry`; QR generation, the card layout, and the share / scan / save actions |
| `utils/SaveImage.ets` | `saveImage(pixelMap, context, uiContext)` - the whole pack-write-dialog-copy chain |
| `utils/ShowError.ets` | `showError(businessError, uiContext)` - a toast for a `BusinessError` |
| `constants/Constants.ets` | layout constants |
| `components/CustomTextBox.ets` | declared but never imported - see HW-05-0028 |

Note the entry file is `EntryAbility.**ts**`, not `.ets`, and `module.json5`
points `srcEntry` at `./ets/entryability/EntryAbility.ts` accordingly.

Data flow for save:

```
save button onClick
  -> uiContext.getComponentSnapshot().get('snapshot', cb, { scale: 0.5 })
       (the id 'snapshot' is set on the Image that renders the QR PixelMap)
  -> saveImage(pixelMap, context, uiContext)
       -> imagePackerApi.packToData(pixelMap, { format: 'image/png', quality: 100 })
       -> fs.openSync(context.filesDir + '/test.png', READ_WRITE | CREATE)
       -> fs.writeSync(fd, buffer)  -> fs.closeSync
       -> fileUri.getUriFromPath(context.filesDir + '/test.png')
       -> photoAccessHelper.getPhotoAccessHelper(context)
            .showAssetsCreationDialog(srcFileUris, photoCreationConfigs)
       -> fileIo.openSync(desFileUris[0], CREATE | READ_WRITE)
       -> fileIo.copyFileSync(srcPath, dstFd) -> fileIo.closeSync
       -> toast '保存成功'
```

Data flow for scan:

```
scan button onClick
  -> scanBarcode.startScanForResult(context, { scanTypes: [ALL], enableMultiMode: true, enableAlbum: true })
  -> result.scanType / result.originalValue -> @State -> toast
```

## Implementation steps

1. **Declare no permission for this feature.** The default scanning UI has the
   camera pre-granted, and `showAssetsCreationDialog` is precisely the
   permission-free save route. The sample's `ohos.permission.CAMERA` entry is an
   over-declaration (HW-05-0022).
2. **Generate the QR code in `aboutToAppear`.** Build a
   `generateBarcode.CreateOptions` with `scanType: scanCore.ScanType.QR_CODE`,
   width/height, `margin`, `level: ErrorCorrectionLevel.LEVEL_H`,
   `backgroundColor` and `pixelMapColor`, call
   `generateBarcode.createBarcode(content, options)` and store the resulting
   `image.PixelMap` in `@State`. Wrap in `try/catch` **and** attach `.catch()` -
   the API can fail both ways.
3. **Render the code with an id.** `Image(this.pixelMap).id('snapshot')` - the id
   is what `getComponentSnapshot().get()` addresses later. Guard the render with
   `if (this.pixelMap)` because the state starts `undefined`.
4. **Snapshot on save.** `this.uiContext.getComponentSnapshot().get('snapshot',
   (error, pixelMap) => {...}, { scale: 0.5 })`. Check `pixelMap` before use; the
   sample already falls back to a `save_error` toast.
5. **Pack to PNG.** `image.createImagePacker().packToData(pixelMap, { format:
   'image/png', quality: 100 })` returns an `ArrayBuffer` through a promise.
6. **Write to the sandbox.** `fs.openSync(context.filesDir + '/test.png',
   fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE)`, `fs.writeSync(file.fd, buffer)`,
   then close **once**, in a guarded `finally` (HW-05-0023).
7. **Raise the save dialog.** Build the sandbox URI with
   `fileUri.getUriFromPath(...)` and a `PhotoCreationConfig` array
   (`fileNameExtension: 'png'`, `photoType: PhotoType.IMAGE`,
   `subtype: PhotoSubtype.DEFAULT`, optional `title`), then
   `photoAccessHelper.getPhotoAccessHelper(context).showAssetsCreationDialog(...)`.
   `await` it inside the surrounding `try/catch`, or attach a `.catch()`; the
   user dismissing the dialog is a rejection (HW-05-0025). For the app name to
   appear in the dialog, `label` and `icon` must be configured under `abilities`
   in `module.json5`.
8. **Copy into the destination.** Open `desFileUris[0]`, `fileIo.copyFileSync(srcPath,
   destFd)`, then close **once** (HW-05-0024) and toast success.
9. **Scan with the default UI.** `scanBarcode.startScanForResult(context,
   options)`; on resolve read `result.scanType` and `result.originalValue`; on
   reject route the `BusinessError` into the project's own `showError` helper
   rather than an empty branch (HW-05-0026).
10. **Handle the insets.** Publish `topRectHeight` / `bottomRectHeight` from
    `EntryAbility`, subscribe to `avoidAreaChange`, and **unsubscribe** in
    `onWindowStageDestroy` (HW-05-0027). Pages read them with `@StorageProp` and
    convert with `px2vp`.

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### QR generation

`Personal_Card.zip#entry/src/main/ets/pages/MainPage.ets`

```ts
import { generateBarcode, scanBarcode, scanCore } from '@kit.ScanKit';
import { image } from '@kit.ImageKit';

mContext: string = '医生王锦涛';                       // QR content
mScanType: number = scanCore.ScanType.QR_CODE;
mWidth: string = '200';
mHeight: string = '200';
mMargin: number = 1;                                  // px
mLevel: generateBarcode.ErrorCorrectionLevel = generateBarcode.ErrorCorrectionLevel.LEVEL_H;
mBackgroundColor: number = 0xffffff;
mPixelMapColor: number = 0x000000;
@State pixelMap: image.PixelMap | undefined = undefined;

aboutToAppear(): void {
  this.pixelMap = undefined;
  let content = this.mContext;
  let options: generateBarcode.CreateOptions = {
    scanType: this.mScanType,
    width: Number(this.mWidth),
    height: Number(this.mHeight),
    margin: Number(this.mMargin),
    level: this.mLevel,
    backgroundColor: this.mBackgroundColor,
    pixelMapColor: this.mPixelMapColor,
  };
  try {
    generateBarcode.createBarcode(content, options).then((result: image.PixelMap) => {
      this.pixelMap = result;
    }).catch((error: BusinessError) => {
      hilog.error(0x0001, TAG,
        `Failed to get pixelMap by promise with options. Code: ${error.code}, message: ${error.message}`);
    });
  } catch (error) {
    showError(error, this.uiContext);
    hilog.error(0x0001, TAG, `Failed to createBarCode. Code: ${error.code}, message: ${error.message}`);
  }
}
```

The rendered component carries the id the snapshot addresses:

```ts
if (this.pixelMap) {
  Image(this.pixelMap).width(190).height(190).objectFit(ImageFit.Contain).id('snapshot');
}
```

### Component snapshot on the save button

`Personal_Card.zip#entry/src/main/ets/pages/MainPage.ets`

```ts
.onClick(() => {
  try {
    this.uiContext.getComponentSnapshot()
      .get('snapshot'/*图片组件绑定的id*/, async (error: Error, pixelMap: image.PixelMap) => {
        if (pixelMap) {
          saveImage(pixelMap, this.context, this.uiContext);
        } else {
          hilog.info(0x0000, 'testTag', '%{public}s', error);
          this.getUIContext().getPromptAction().showToast({ message: $r('app.string.save_error') });
        }
      }, { scale: 0.5 });
  } catch (error) {
    this.getUIContext().getPromptAction().showToast({ message: $r('app.string.save_error') });
  }
});
```

### Pack, sandbox write, authorization dialog, copy

`Personal_Card.zip#entry/src/main/ets/utils/SaveImage.ets`

```ts
import { photoAccessHelper } from '@kit.MediaLibraryKit';
import { fileIo, fileUri } from '@kit.CoreFileKit';
import fs from '@ohos.file.fs';
import { image } from '@kit.ImageKit';

export async function saveImage(pixelMap: PixelMap, context: common.UIAbilityContext, uiContext: UIContext) {
  const imagePackerApi: image.ImagePacker = image.createImagePacker();
  let packOptions: image.PackingOption = {
    format: 'image/png',
    quality: 100
  };
  imagePackerApi.packToData(pixelMap, packOptions).then((buffer: ArrayBuffer) => {
    let path = context.filesDir + '/test.png';
    let file: fileIo.File = null!;
    try {
      file = fs.openSync(path, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
      fs.writeSync(file.fd, buffer);
      fs.closeSync(file.fd);                       // first close - see HW-05-0023

      let srcFileUris: Array<string> = [fileUri.getUriFromPath(context.filesDir + '/test.png')];
      let photoCreationConfigs: Array<photoAccessHelper.PhotoCreationConfig> = [
        {
          title: 'test',
          fileNameExtension: 'png',
          photoType: photoAccessHelper.PhotoType.IMAGE,
          subtype: photoAccessHelper.PhotoSubtype.DEFAULT,
        },
      ];
      photoAccessHelper.getPhotoAccessHelper(context)
        .showAssetsCreationDialog(srcFileUris, photoCreationConfigs)
        .then((desFileUris: Array<string>) => {     // no .catch - see HW-05-0025
          let imageFile1 = fileIo.openSync(desFileUris[0], fileIo.OpenMode.CREATE | fileIo.OpenMode.READ_WRITE);
          let srcPath = context.filesDir + '/test.png';
          let dstPath = imageFile1.fd;
          try {
            fileIo.copyFileSync(srcPath, dstPath);
            fileIo.closeSync(imageFile1.fd);        // first close - see HW-05-0024
            uiContext.getPromptAction().showToast({ message: '保存成功' });
          } catch (error) {
            hilog.error(0x0000, 'testTag', `Failed to saveFile. Code: ${error.code}, message: ${error.message}`);
          } finally {
            if (imageFile1) {
              fileIo.closeSync(imageFile1);         // second close
            }
          }
        });
    } catch (error) {
      hilog.error(0x0000, 'testTag', `Failed to saveFile. Code: ${error.code}, message: ${error.message}`);
    } finally {
      fs.closeSync(file.fd);                        // second close, and a TypeError if openSync threw
    }
  }).catch((error: BusinessError) => {
    hilog.error(0x0000, 'testTag', `Failed to saveFile. Code: ${error.code}, message: ${error.message}`);
  });
}
```

Corrected file handling, following the official save-user-file example:

```ts
let file: fileIo.File | undefined = undefined;
try {
  file = fs.openSync(path, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
  fs.writeSync(file.fd, buffer);
} finally {
  if (file) {
    fs.closeSync(file);          // exactly once, only when open succeeded
  }
}

try {
  const desFileUris = await photoAccessHelper.getPhotoAccessHelper(context)
    .showAssetsCreationDialog(srcFileUris, photoCreationConfigs);
  let desFile: fileIo.File | undefined = undefined;
  try {
    desFile = fileIo.openSync(desFileUris[0], fileIo.OpenMode.WRITE_ONLY);
    fileIo.copyFileSync(srcPath, desFile.fd);
    uiContext.getPromptAction().showToast({ message: '保存成功' });
  } finally {
    if (desFile) {
      fileIo.closeSync(desFile);
    }
  }
} catch (err) {
  hilog.error(0x0000, 'testTag', 'save cancelled or failed: %{public}s', JSON.stringify(err));
  uiContext.getPromptAction().showToast({ message: $r('app.string.save_error') });
}
```

### Scanning another person's code

`Personal_Card.zip#entry/src/main/ets/pages/MainPage.ets`

```ts
scanBarCode() {
  let options: scanBarcode.ScanOptions = {
    scanTypes: [scanCore.ScanType.ALL],
    enableMultiMode: true,
    enableAlbum: true
  };
  try {
    scanBarcode.startScanForResult(this.context, options).then((result: scanBarcode.ScanResult) => {
      this.scanType = result.scanType;
      this.inputValue = result.originalValue;
      this.getUIContext().getPromptAction().showToast({ message: this.inputValue });
    }).catch((error: BusinessError) => {
      if (error.code === scanCore.ScanErrorCode.INTERNAL_ERROR) {   // empty - see HW-05-0026
      } else {
      }
    });
  } catch (error) {
  }
}
```

Corrected failure path, reusing the helper the project already ships:

```ts
}).catch((error: BusinessError) => {
  hilog.error(0x0001, TAG, `scan failed. Code: ${error.code}, message: ${error.message}`);
  showError(error, this.uiContext);
});
} catch (error) {
  showError(error, this.uiContext);
}
```

### The unused error-toast helper

`Personal_Card.zip#entry/src/main/ets/utils/ShowError.ets`

```ts
export function showError(businessError: BusinessError, uiContext: UIContext): void {
  try {
    uiContext.getPromptAction().showToast({
      message: `Error Code: ${businessError.code}, message: ${businessError.message}`,
      duration: Constants.DURATION
    });
  } catch (error) {
    let message = (error as BusinessError).message;
    let code = (error as BusinessError).code;
    hilog.error(0x0001, 'testTag', `Failed to ShowToast. Code: ${code}, message: ${message}`);
  }
}
```

## Permissions & config

`Personal_Card.zip#entry/src/main/module.json5` - as shipped:

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["default"],
    "pages": "$profile:main_pages",
    "abilities": [
      {
        "name": "EntryAbility",
        "srcEntry": "./ets/entryability/EntryAbility.ts",
        "icon": "$media:startIcon",
        "label": "$string:EntryAbility_label",
        "exported": true,
        "skills": [
          { "entities": ["entity.system.home"], "actions": ["action.system.home"] }
        ]
      }
    ],
    "requestPermissions": [
      {
        // over-declared - see HW-05-0022
        "name": "ohos.permission.CAMERA",
        "reason": "$string:permission_reason_camera",
        "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
      }
    ]
  }
}
```

Recommended form for this feature: **no `requestPermissions` block at all**.

- The Scan Kit default UI has the camera pre-granted at system level.
- `showAssetsCreationDialog` grants a per-file write authorization through the
  dialog, so no media-library permission is needed.
- `label` and `icon` under `abilities` are still required: the save dialog shows
  the application name from them, and without them the name is blank.
- `deviceTypes` is `["default"]` here, unlike the other office samples which list
  `phone`, `tablet` and `2in1`.
- `build-profile.json5` pins `compatibleSdkVersion` and `targetSdkVersion` to
  `6.0.0(20)`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later; the sample pins `6.0.0(20)`.
- **The save dialog is the authorization.** `showAssetsCreationDialog` returns
  destination URIs only after the user agrees; there is no silent path, and the
  returned URIs are single-use write targets, not general gallery access.
- **Sandbox first.** The source of the save must be a file in the application
  sandbox; the guide states that if the passed URI is a sandbox path the image
  can be saved but not previewed in the dialog.
- **The snapshot targets a rendered component.** `getComponentSnapshot().get(id)`
  only works while the component with that id is laid out, which is why the QR
  `Image` is rendered before the save button can be pressed. The sample captures
  at `scale: 0.5`.
- **Gallery scanning is single-code.** `enableAlbum: true` opens the gallery
  entry of the default UI, and the Scan Kit guide notes that gallery-based
  scanning recognises only one barcode at a time even when `enableMultiMode` is
  set.
- **The default scanning UI cannot be customised.** For a custom preview the
  scenario would have to move to `scan-customscan-api`, at which point
  `ohos.permission.CAMERA` genuinely becomes necessary.
- **Card content is hard-coded.** `mContext` is the literal `'医生王锦涛'`; there is
  no account or profile source in this sample.

## Pitfalls

- **Declaring `ohos.permission.CAMERA` here is incorrect.** The Scan Kit guide
  states the camera permission is pre-granted for the default UI and "you do not
  need to request the camera permission again"; the sample uses no other camera
  API. Remove the declaration and the 权限说明 section. (HW-05-0022)
- **`finally { fs.closeSync(file.fd); }` after an in-try close is incorrect.** It
  closes the descriptor a second time, and when `openSync` itself threw, `file`
  is still the `null!` it was initialised with, so the finally raises a TypeError
  that replaces the real error. Close once, guarded: `if (file) { fs.closeSync(file); }`.
  (HW-05-0023)
- **Closing the destination file both by fd and by File object is incorrect.**
  `fileIo.closeSync(imageFile1.fd)` at `:60` and `fileIo.closeSync(imageFile1)` at
  `:66` release the same descriptor twice. Keep only the `finally` close.
  (HW-05-0024)
- **`showAssetsCreationDialog(...).then(...)` with no `.catch()` is incorrect.**
  Dismissing the dialog is an ordinary outcome and rejects the promise; the
  enclosing synchronous `try/catch` cannot see it. `await` it inside the
  `try/catch`, as the official guide does. (HW-05-0025)
- **Two empty `if/else` branches plus an empty `catch` on the scan path are
  incorrect.** The project already ships `showError(businessError, uiContext)` and
  uses it for barcode generation; route the scan failure through the same helper.
  (HW-05-0026)
- **Subscribing to `avoidAreaChange` without `off` is incorrect.** Unsubscribe in
  `onWindowStageDestroy`. (HW-05-0027)
- **The project tree calls `CustomTextBox.ets` the scan-result display, which is
  incorrect.** Nothing imports it; the result is shown with a toast at
  `MainPage.ets:95`. Either wire the component in or correct the tree comment.
  (HW-05-0028)
- **`hilog.info(0x0000, 'testTag', JSON.stringify(want))` is incorrect.** The
  serialized object becomes the format string, so `%` sequences are treated as
  specifiers and the content is filtered as private data. Use
  `hilog.info(0x0000, 'testTag', 'want: %{public}s', JSON.stringify(want))`.
  (HW-05-0029)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-guides/02_media/scan-scanbarcode.md` - the default
  scanning UI, the pre-granted camera permission, `startScanForResult` overloads
  and the single-code gallery restriction.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/scan-scanbarcode
- `documentation/harmonyos-guides/05_media/photoaccesshelper-savebutton.md` -
  the `showAssetsCreationDialog` save flow, the `label`/`icon` requirement, and
  the reference example that awaits inside `try/catch` and closes each handle
  once.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-savebutton
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `fileIo.closeSync(file: number | File)` and "Once closed, the File object or FD
  cannot be used for read or write operations."
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/03_system/js-apis-hilog.md` -
  `info(domain, tag, format, ...args)` and the `{public}` / `{private}` rules.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `on('avoidAreaChange')` / `off('avoidAreaChange')`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - the rule
  that the stated reason must match the actual permission function.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
