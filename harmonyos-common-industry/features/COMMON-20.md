---
id: COMMON-20
title: Screenshot detection and sharing - listen for the system screenshot, snapshot the page, save it to the album or share it
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/20_screenshot_listen.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/screenshot_listen-0000002261676460
sample: huawei_industry_tree/19_common_technical_solutions/downloads/ScreenshotSharing.zip
kits: ["@kit.ArkUI", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.CoreFileKit", "@kit.ShareKit", "@kit.ArkData", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.LocationKit"]
apis: ["window.on('screenshot')", "window.off('screenshot')", "UIContext.getComponentSnapshot", "ComponentSnapshot.get", CustomDialog, CustomDialogController, "image.createImagePacker", "ImagePacker.packToData", "image.PackingOption", "PixelMap.release", "fileIo.openSync", "fileIo.writeSync", "fileIo.closeSync", "fileIo.copyFileSync", "fileUri.getUriFromPath", "photoAccessHelper.getPhotoAccessHelper", "PhotoAccessHelper.showAssetsCreationDialog", "photoAccessHelper.PhotoCreationConfig", "systemShare.SharedData", "systemShare.ShareController", "ShareController.show", "systemShare.SelectionMode", "systemShare.SharePreviewMode", "systemShare.ShareAbilityType", "uniformTypeDescriptor.getUniformDataTypeByFilenameExtension", "utd.UniformDataType"]
permissions: []
min_api: 20
modules: [entry, common]
findings: [HW-19-0058, HW-19-0059, HW-19-0060, HW-19-0061, HW-19-0062, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when the application should **react to the user taking a system
screenshot** - pop a prompt offering to save a nicer capture to the album or to
share it. The document's framing: 应用检测到用户截屏操作，会弹出分享框提示 ("when the
application detects the user's screenshot action, it pops up a share prompt").

Three separable capabilities sit behind it, and each is reusable on its own:
detecting the screenshot event, turning a component subtree into a `PixelMap`,
and getting that image into the album or into the system share sheet.

**The feature itself needs no permission.** Saving goes through
`showAssetsCreationDialog`, which is a system save dialog, so no media-write
permission is required (HW-19-0061 covers the document's missing permission
section for the surrounding demo).

## Feature checklist

The application must:

- Subscribe to the system screenshot event on the page's window, and unsubscribe
  when the page goes away.
- Capture the page as a `PixelMap` - note that this is an application-side
  snapshot of its own component tree, not the system screenshot image.
- Show a dialog previewing that capture, with save and share actions
  (HW-19-0062).
- Encode the `PixelMap` to PNG bytes and write them to a sandbox file, closing
  the file on every path (HW-19-0058, HW-19-0059).
- Save through `showAssetsCreationDialog` and copy the bytes into the URI it
  returns.
- Share through `systemShare.ShareController` with a precise UTD type derived
  from the file extension.
- Release each `PixelMap` once it is no longer needed (HW-19-0060).

## Architecture

Two modules: `entry` (the product) and `common` (a HAR carrying the shopping-demo
shell this feature is embedded in).

| File | Responsibility |
| --- | --- |
| `entry/.../entryability/EntryAbility.ets` | publishes `windowClass` (the main window) and `currentUIContext` into `AppStorage` |
| `entry/.../pages/Index.ets` | the `shareDialog` `@CustomDialog`, the `Index` page, the `screenshot` subscription and the `getComponentSnapshot` capture |
| `entry/.../utils/SaveImage.ets` | `saveImage(pixelMap)` - pack to PNG, write to sandbox, `showAssetsCreationDialog`, copy in |
| `entry/.../utils/ShareOperations.ets` | `handelShare(context)` for a link and `handelShareImage(pixelMap)` for the capture |
| `entry/.../constants/CommodityConstants.ets` | `IMAGE_URL = '/test.png'`, the sandbox file name |
| `entry/.../components/CommodityDetail.ets` | the product page being screenshotted; also the only consumer of Location Kit |
| `common/...` (HAR) | the shopping shell: components, view models, `Logger`, `LocationManager`, breakpoints |

**Detection flow.**

1. `EntryAbility.onWindowStageCreate` stores `windowStage.getMainWindowSync()` in
   `AppStorage` under `windowClass`, and the main window's `UIContext` under
   `currentUIContext` - the latter is how the two util files reach a context from
   outside a component.
2. `Index.aboutToAppear` calls `this.windowClass.on('screenshot', cb)` inside a
   `try/catch`.
3. `Index.aboutToDisappear` calls `this.windowClass.off('screenshot')`.
4. The callback captures the page and opens the dialog:
   ```ts
   this.pixmap = await this.screenshot();
   this.shareDialogController?.close();
   this.shareDialogController?.open();
   ```
   The `close()` before `open()` is deliberate - it drops a dialog left over from
   a previous screenshot.

**Capture.** `screenshot()` wraps
`uiContext.getComponentSnapshot().get('root', cb, { scale: 2,
waitUntilRenderFinished: true })` in a promise. `'root'` is the `id` set on the
page's outermost `Row`, so the capture is the application's own rendering of the
page - it deliberately does not include system UI, and it can be styled
differently from what the user actually screenshotted.

**Save.** `packToData` -> sandbox file -> `showAssetsCreationDialog(srcFileUris,
photoCreationConfigs)` -> open the returned destination URI ->
`copyFileSync(srcPath, dstFd)`. The dialog is what grants the write, which is why
no media permission appears anywhere.

**Share.** `packToData` -> sandbox file ->
`utd.getUniformDataTypeByFilenameExtension('.png', utd.UniformDataType.IMAGE)` ->
`SharedData({ utd, uri, title, description })` -> `ShareController.show(context,
{ selectionMode: SINGLE, previewMode: DETAIL })`.

## Implementation steps

1. **Publish the window and UI context** in `onWindowStageCreate` so page code and
   non-component utilities can reach them.
2. **Subscribe on appear, unsubscribe on disappear.**
   `windowClass.on('screenshot', cb)` in `aboutToAppear` inside `try/catch`;
   `windowClass.off('screenshot')` in `aboutToDisappear`.
3. **Give the capture root an `id`** and snapshot it with
   `getComponentSnapshot().get(id, cb, { scale, waitUntilRenderFinished: true })`.
4. **Bind the dialog preview to state, not to a copied value** - `@Link` plus
   `$pixmap`, or build the controller when the screenshot arrives (HW-19-0062).
5. **Encode once, reuse the file.** `image.createImagePacker()`,
   `packToData(pixelMap, { format: 'image/png', quality: 100 })`.
6. **Write the sandbox file with a `finally`**: open, write in `try`, close in
   `finally`, exactly once (HW-19-0058, HW-19-0059).
7. **Save via the system dialog**: `getPhotoAccessHelper(context)
   .showAssetsCreationDialog(srcFileUris, photoCreationConfigs)`, then open
   `desFileUris[0]` and `copyFileSync` into it.
8. **Share with a precise UTD type** rather than the generic image type.
9. **Release the PixelMap** when replacing it and on page exit (HW-19-0060).

## Verified snippets

All snippets below come from the sample project, not from the document.

### Subscribing to the system screenshot

`ScreenshotSharing.zip#ScreenshotSharing/entry/src/main/ets/pages/Index.ets`

```ts
@Entry
@Component
struct Index {
  commodityId: string = '1';
  windowClass: window.Window = AppStorage.get('windowClass') as window.Window;
  @State pixmap: image.PixelMap | undefined = undefined;
  private uiContext: UIContext = this.getUIContext();

  aboutToAppear(): void {
    try {
      this.windowClass.on('screenshot', async () => {
        this.pixmap = await this.screenshot();   // FIX (HW-19-0060): release the previous one
        this.shareDialogController?.close();
        this.shareDialogController?.open();
        Logger.info('screenshot happened');
      });
    } catch (exception) {
      Logger.error(`Failed to register callback. Cause code: ${exception.code}, message: ${exception.message}`);
    }
  }

  aboutToDisappear(): void {
    this.windowClass.off('screenshot');
  }

  screenshot(): Promise<image.PixelMap> {
    return new Promise((resolve, reject) => {
      this.uiContext.getComponentSnapshot().get('root', (error: Error, pixmap: image.PixelMap) => {
        if (error) {
          Logger.error('Snapshot-error: ' + JSON.stringify(error));
          reject(error);
          return;
        }
        resolve(pixmap);
      }, { scale: 2, waitUntilRenderFinished: true });
    });
  }

  build() {
    Row() {
      CommodityDetail({ commodityId: this.commodityId });
    }
    .backgroundColor($r('app.color.page_background'))
    .id('root');            // the snapshot target
  }
}
```

### The share dialog (as shipped - see HW-19-0062)

`ScreenshotSharing.zip#ScreenshotSharing/entry/src/main/ets/pages/Index.ets`

```ts
@CustomDialog
struct shareDialog {
  controller?: CustomDialogController;
  pixmap: image.PixelMap | undefined = undefined;   // FIX: @Link pixmap
  save: () => void = () => {};
  share: () => void = () => {};

  build() {
    Column() {
      Row()
        .width('100%')
        .height('420lpx')
        .backgroundImage(this.pixmap)
        .backgroundImageSize({ width: '100%' })
        .backgroundImagePosition(Alignment.TopStart);
      // ... save / share buttons ...
    };
  }
}

shareDialogController: CustomDialogController | null = new CustomDialogController({
  builder: shareDialog({
    save: () => {
      if (this.pixmap) {
        saveImage(this.pixmap);
      } else {
        this.getUIContext().getPromptAction().showToast({ message: '图片获取失败！' });
      }
    },
    share: () => {
      if (this.pixmap) {
        ShareOperations.handelShareImage(this.pixmap);
      } else {
        this.getUIContext().getPromptAction().showToast({ message: '图片获取失败！' });
      }
    },
    pixmap: this.pixmap        // FIX (HW-19-0062): pass $pixmap to a @Link
  }),
  cancel: () => {},
  autoCancel: true,
  alignment: DialogAlignment.Center,
  customStyle: false,
  maskColor: 'rgba(0, 0, 0, 0.5)',
  cornerRadius: 8,
  width: '293lpx',
  backgroundColor: Color.White,
});
```

### Saving to the album (as shipped - see HW-19-0059)

`ScreenshotSharing.zip#ScreenshotSharing/entry/src/main/ets/utils/SaveImage.ets`

```ts
export async function saveImage(pixelMap: PixelMap) {
  const uiContext: UIContext | undefined = AppStorage.get('currentUIContext');
  const imagePackerApi: image.ImagePacker = image.createImagePacker();
  let packOptions: image.PackingOption = {
    format: 'image/png',
    quality: 100
  };
  imagePackerApi.packToData(pixelMap, packOptions).then((buffer: ArrayBuffer) => {
    const context = uiContext?.getHostContext();
    // 应用沙箱路径
    let path = context?.filesDir + CommodityConstants.IMAGE_URL;
    // 在沙箱新建并打开文件
    let file = fileIo.openSync(path, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
    try {
      // 写入pixelMap图片内容
      fileIo.writeSync(file.fd, buffer);
      // 关闭文件
      fileIo.closeSync(file.fd);      // FIX (HW-19-0059): move to a finally block
      // 使用showAssetsCreationDialog保存沙箱中的图片
      let srcFileUris: Array<string> = [fileUri.getUriFromPath(context?.filesDir + CommodityConstants.IMAGE_URL)];
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
        .then((desFileUris: Array<string>) => {
          let imageFile1 = fileIo.openSync(desFileUris[0], fileIo.OpenMode.CREATE | fileIo.OpenMode.READ_WRITE);
          let srcPath = context?.filesDir + CommodityConstants.IMAGE_URL;
          let dstPath = imageFile1.fd;
          try {
            fileIo.copyFileSync(srcPath, dstPath);
            fileIo.closeSync(imageFile1.fd);
            uiContext?.getPromptAction().showToast({ message: '保存成功' });
          } catch (error) {
            uiContext?.getPromptAction().showToast({ message: '保存失败' });
            Logger.error(`Failed to saveFile1. Code: ${error.code}, message: ${error.message}`);
          }
        }).catch((error: BusinessError) => {
          uiContext?.getPromptAction().showToast({ message: '保存失败' });
          Logger.error(`Failed to showAssetsCreationDialog. Code: ${error.code}, message: ${error.message}`);
        });
    } catch (error) {
      uiContext?.getPromptAction().showToast({ message: '保存失败' });
      Logger.error(`Failed to saveFile2. Code: ${error.code}, message: ${error.message}`);
    }
  }).catch((error: BusinessError) => {
    uiContext?.getPromptAction().showToast({ message: '保存失败' });
    Logger.error(`Failed to saveFile3. Code: ${error.code}, message: ${error.message}`);
  });
}
```

### Sharing the image (as shipped - see HW-19-0058)

`ScreenshotSharing.zip#ScreenshotSharing/entry/src/main/ets/utils/ShareOperations.ets`

```ts
static async handelShareImage(pixelMap: PixelMap): Promise<void> {
  const uiContext: UIContext | undefined = AppStorage.get('currentUIContext');
  const context = uiContext?.getHostContext() as common.UIAbilityContext;
  const imagePackerApi: image.ImagePacker = image.createImagePacker();
  let packOptions: image.PackingOption = { format: 'image/png', quality: 100 };
  imagePackerApi.packToData(pixelMap, packOptions).then((buffer: ArrayBuffer) => {
    // 应用沙箱路径
    let filePath = context.filesDir + CommodityConstants.IMAGE_URL;
    // 在沙箱新建并打开文件
    let file = fileIo.openSync(filePath, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
    try {
      fileIo.writeSync(file.fd, buffer);
      fileIo.closeSync(file.fd);        // FIX (HW-19-0058): double close with the finally below
      // 获取精准的utd类型
      let utdTypeId = utd.getUniformDataTypeByFilenameExtension('.png', utd.UniformDataType.IMAGE);
      let shareData: systemShare.SharedData = new systemShare.SharedData({
        utd: utdTypeId,
        uri: fileUri.getUriFromPath(filePath),
        title: '图片标题', // 不传title字段时,显示图片文件名
        description: '图片描述', // 不传description字段时,显示图片大小
      });
      // 进行分享面板显示
      let controller: systemShare.ShareController = new systemShare.ShareController(shareData);
      controller.show(context, {
        selectionMode: systemShare.SelectionMode.SINGLE,
        previewMode: systemShare.SharePreviewMode.DETAIL,
      }).then(() => {
        Logger.info('ShareController show success.');
      }).catch((error: BusinessError) => {
        Logger.error(`ShareController show error. code: ${error.code}, message: ${error.message}`);
      });
    } catch (error) {
      uiContext?.getPromptAction().showToast({ message: '分享失败' });
      Logger.error(`Failed to saveFile2. Code: ${error.code}, message: ${error.message}`);
    } finally {
      fileIo.closeSync(file);
    }
  }).catch((error: BusinessError) => { /* ... */ });
}
```

### Sharing a link (the non-image path)

`ScreenshotSharing.zip#ScreenshotSharing/entry/src/main/ets/utils/ShareOperations.ets`

```ts
static async handelShare(context: common.UIAbilityContext): Promise<void> {
  let shareData: systemShare.SharedData = new systemShare.SharedData({
    utd: utd.UniformDataType.HYPERLINK,
    content: context.resourceManager.getStringSync($r('app.string.share_url').id),
    title: '华为商城',
    description: 'Pura 70 Ultra',
    label: '华为商城' // 单选模式时生效
  });
  let controller: systemShare.ShareController = new systemShare.ShareController(shareData);
  controller.show(context, {
    excludedAbilities: [systemShare.ShareAbilityType.COPY_TO_PASTEBOARD], // 从操作区排除复制操作
  });
}
```

### Publishing the window and UI context

`ScreenshotSharing.zip#ScreenshotSharing/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
let windowClass: window.Window = windowStage.getMainWindowSync();
AppStorage.setOrCreate('windowClass', windowClass);
// ...
AppStorage.setOrCreate('currentUIContext', windowStage.getMainWindowSync().getUIContext());
```

## Permissions & config

**The screenshot / save / share feature needs no permission.**
`window.on('screenshot')` is a window event on the application's own window,
`getComponentSnapshot` captures the application's own component tree, and
`showAssetsCreationDialog` is a system save dialog that grants the write for the
files the user confirms - which is why no media permission appears anywhere in
the project.

The two permissions the sample **does** declare belong to the shopping shell it is
built on, not to this feature - the document never mentions them (HW-19-0061):

`ScreenshotSharing.zip#ScreenshotSharing/entry/src/main/module.json5`:

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.LOCATION",
    "reason": "$string:reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  },
  {
    "name": "ohos.permission.APPROXIMATELY_LOCATION",
    "reason": "$string:reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
]
```

They are consumed by
`ScreenshotSharing.zip#ScreenshotSharing/common/src/main/ets/utils/LocationManager.ets`
and `entry/src/main/ets/components/CommodityDetail.ets`, which reverse-geocodes
the shipping address. Drop both when lifting the screenshot feature out.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. `window.on('screenshot')` dates
  from API 9; `ImagePacker.packToData` from API 13.
- **The event tells you a screenshot happened; it does not give you the image.**
  The system screenshot bitmap is not delivered to the application - the sample
  produces its own capture with `getComponentSnapshot`, which is why the preview
  can differ from what the user actually captured.
- **`getComponentSnapshot().get(id, ...)` needs the target to carry that `id`**;
  the page sets `.id('root')` on its outermost `Row`.
- **`showAssetsCreationDialog` returns destination URIs the app may write to.**
  Handle an empty array - the user can cancel - before indexing `desFileUris[0]`.
- **A closed file object or FD cannot be reused**: "Once closed, the File object
  or FD cannot be used for read or write operations."
- **PixelMaps must be released manually** once no longer needed, per the image
  guide.
- **`utd.getUniformDataTypeByFilenameExtension`** gives the precise type; passing
  the generic `UniformDataType.IMAGE` loses the PNG specificity the share sheet
  uses for previews.
- **Devices.** `phone`, `tablet`, `2in1` per `module.json5`.

## Pitfalls

- **`handelShareImage` closes the file twice, which is incorrect** -
  `closeSync(file.fd)` inside the `try` and `closeSync(file)` in the `finally`.
  Keep only the `finally`. (HW-19-0058)
- **`saveImage` has no `finally`, which is incorrect.** The close is the second
  statement in the `try`, so a failing `writeSync` leaks the descriptor - once per
  screenshot. (HW-19-0059)
- **No `PixelMap` is ever released, which is incorrect.** Each screenshot
  allocates a fresh full-page bitmap at `scale: 2` and the previous one is
  dropped without `release()`. (HW-19-0060)
- **The dialog preview is passed by value at controller-construction time, which
  is incorrect.** `pixmap: this.pixmap` copies `undefined`; the preview never
  updates even though the save/share closures see the current value. Use `@Link`
  + `$pixmap`. (HW-19-0062)
- **The document has no 权限说明 section** although the sample declares two
  user-grant location permissions - which, confusingly, this feature does not
  need at all. (HW-19-0061)
- **`shareDialogController?.close()` before `open()` is intentional.** It clears a
  dialog left over from an earlier screenshot; without it a second screenshot
  while the first prompt is open does nothing visible.
- **`AppStorage.get('currentUIContext')` is how the util files reach a context.**
  That coupling means `saveImage` and `handelShareImage` silently no-op if the
  ability has not populated `AppStorage` yet - both use `uiContext?.` throughout
  and never report the undefined case.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `on('screenshot')` / `off('screenshot')`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-window-window#onscreenshot9
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `openSync`, `writeSync`, `closeSync(file: number | File)` and the
  once-closed-unusable rule, `copyFileSync`, `OpenMode`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-guides/02_media/image-decoding.md` - "Release the
  PixelMap and ImageSource instances ... you can manually call the APIs below to
  release them."
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-decoding
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-image-imagepacker#packtodata13-1 -
  `ImagePacker.packToData` and `PackingOption`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-methods-custom-dialog-box -
  `@CustomDialog` and `CustomDialogController`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/share-system-share -
  `systemShare.SharedData`, `ShareController.show`, `SelectionMode`,
  `SharePreviewMode`, `ShareAbilityType`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/share-utd-image -
  sharing an image with a UTD type.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faqs-image-20 -
  saving a PixelMap to the album, the pattern `saveImage` follows.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/screenshot_listen-0000002261676460
