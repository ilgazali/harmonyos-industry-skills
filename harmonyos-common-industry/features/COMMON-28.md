---
id: COMMON-28
title: H5 long-press on an image - custom context menu to save the picture to the album or copy its link
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/28_press_to_save_or_copy.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/press_to_save_or_copy-0000002328179937
sample: huawei_industry_tree/19_common_technical_solutions/downloads/PressToSaveOrCopy.zip
kits: ["@kit.ArkWeb", "@kit.MediaLibraryKit", "@kit.CoreFileKit", "@kit.ArkUI", "@kit.AbilityKit", "@kit.BasicServicesKit"]
apis: ["Web.onContextMenuShow", "WebContextMenuParam.getMediaType", "WebContextMenuParam.getSourceUrl", "WebContextMenuParam.y", "WebContextMenuResult.closeContextMenu", ContextMenuMediaType, "component .bindPopup", Placement, "webview.WebDownloadDelegate", "WebDownloadDelegate.onBeforeDownload", "WebDownloadDelegate.onDownloadUpdated", "WebDownloadDelegate.onDownloadFailed", "WebDownloadDelegate.onDownloadFinish", "WebDownloadItem.start", "WebDownloadItem.getSuggestedFileName", "WebDownloadItem.getGuid", "WebDownloadItem.getPercentComplete", "WebDownloadItem.getCurrentSpeed", "WebDownloadItem.getLastErrorCode", "WebviewController.setDownloadDelegate", "WebviewController.startDownload", "photoAccessHelper.getPhotoAccessHelper", "PhotoAccessHelper.showAssetsCreationDialog", "photoAccessHelper.PhotoCreationConfig", "fileIo.open", "fileIo.copyFile", "fileIo.closeSync", "fileUri.getUriFromPath", "pasteboard.createData", "pasteboard.getSystemPasteboard", "SystemPasteboard.setData", "PromptAction.showToast", "UIContext.px2lpx"]
permissions: ["ohos.permission.INTERNET"]
min_api: 20
modules: [entry]
findings: [HW-19-0087, HW-19-0088, HW-19-0089, HW-19-0182, HW-19-0183]
status: verified-with-fixes
---

## When to use

Load this card when an embedded H5 page shows images and the user should be able
to **long-press one to save it or copy its link** - the behaviour every browser
offers, implemented by the host application rather than by the page.

Three separable pieces: intercepting the long press and identifying the image,
drawing your own menu at the touch point, and getting the remote image into the
gallery through the system authorisation dialog.

## Feature checklist

The application must:

- Intercept the web view's context menu with `onContextMenuShow` and return
  `true` so the built-in menu is suppressed.
- Act only on image long-presses (`ContextMenuMediaType.Image`), and let
  everything else fall through.
- Capture the image URL and the touch position from the event parameters.
- Keep the `WebContextMenuResult` so the menu can be closed properly.
- Draw the menu with `bindPopup`, positioned at the touch point.
- Close the context menu when the popup hides.
- **Save**: download the remote image into the sandbox with the web download
  delegate, then hand it to `showAssetsCreationDialog` - as a **URI**
  (HW-19-0087) - and copy the bytes into the returned media URI.
- **Copy**: put the URL on the system pasteboard and confirm with a toast.
- Report a failed download to the user, not only to the log (HW-19-0088).
- Not grant the page file-system access it does not need (HW-19-0089).

## Architecture

Single-module project (`entry` HAP), one page plus two utilities:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup |
| `utils/Constants.ets` | menu button width/height and divider metrics |
| `utils/Logger.ets` | `hilog` wrapper |
| `pages/Index.ets` | everything: the `Web`, the context-menu interception, the popup menu builder, `savePhotoToGallery` and `showToast` |
| `resources/rawfile/index.html` | the page - a product gallery whose images are all remote https URLs |

**Interception.** `onContextMenuShow((event) => ...)` receives an event carrying
a `param` and a `result`. The handler filters on
`event.param.getMediaType() === ContextMenuMediaType.Image`, stores
`event.result` and `event.param.getSourceUrl()`, computes the popup offset from
`event.param.y()`, sets `showMenu = true` and returns `true`. Returning `true` is
what suppresses the system menu; returning `false` for non-image targets lets the
default behaviour stand.

**The menu is a popup on the `Web` component itself** - `bindPopup(this.showMenu,
{ builder: this.customMenu(), placement: Placement.LeftTop, offset: { x, y },
enableArrow: false, mask: false })`. Its `onStateChange` handles dismissal from
any source: when the popup becomes invisible it clears `showMenu` and calls
`this.result!.closeContextMenu()`, which is what releases the web engine's
context-menu state.

**Save is a four-step pipeline**, and only the first step is Web-specific:

1. `webVC.setDownloadDelegate(delegate)` + `webVC.startDownload(imgUrl)`.
2. `onBeforeDownload` chooses the destination -
   `applicationContext.tempDir + '/' + item.getSuggestedFileName()` - and calls
   `item.start(path)`. The download does not begin until `start` is called.
3. `onDownloadFinish` calls `savePhotoToGallery(uiContext, imgSavedPath)`.
4. `savePhotoToGallery` runs the standard save-button flow:
   `getPhotoAccessHelper` -> `showAssetsCreationDialog(srcFileUris, configs)` ->
   `fileIo.open` both sides -> `fileIo.copyFile` -> close both.

**Copy is one call**: `pasteboard.createData(MIMETYPE_TEXT_PLAIN, imgUrl)` then
`getSystemPasteboard().setData(data, cb)`.

## Implementation steps

1. **Load the page** and keep a `WebviewController`.
2. **Intercept the context menu**, filter on media type, capture URL, result and
   position, and return `true` only for the case you handle.
3. **Position the popup** from `param.y()`; the sample converts with `px2lpx` and
   offsets upward so the menu sits above the finger.
4. **Bind the popup to the `Web`** and give it an `onStateChange` that clears the
   state and calls `closeContextMenu()`.
5. **Create one `WebDownloadDelegate`** and register its four callbacks;
   `onBeforeDownload` must call `start(path)` or the download never runs.
6. **Convert the sandbox path to a URI** before `showAssetsCreationDialog`
   (HW-19-0087).
7. **Copy the bytes** into `desFileUris[0]` and close both files.
8. **Toast every outcome** - success, save failure and download failure
   (HW-19-0088).
9. **Declare `ohos.permission.INTERNET`** (the images and the download are
   remote) and leave `fileAccess` off (HW-19-0089).

## Verified snippets

All snippets below come from the sample project, not from the document.

### Intercepting the long press and drawing the menu

`PressToSaveOrCopy.zip#PressToSaveOrCopy/entry/src/main/ets/pages/Index.ets`

```ts
@Entry
@Component
struct Index {
  private webVC: webview.WebviewController = new webview.WebviewController();
  private imgUrl: string = '';
  private uiContext: UIContext = this.getUIContext();
  private delegate: webview.WebDownloadDelegate = new webview.WebDownloadDelegate();
  private result: WebContextMenuResult | undefined = undefined;
  @State offsetX: number = 0;
  @State offsetY: number = 0;
  @State showMenu: boolean = false;

  build() {
    Column() {
      Web({
        src: $rawfile('index.html'),
        controller: this.webVC
      })
        .fileAccess(true)              // FIX (HW-19-0089): not needed here
        .geolocationAccess(false)
        .onContextMenuShow((event) => {
          // 检测到长按图片，弹出菜单
          if (event?.param.getMediaType() === ContextMenuMediaType.Image) {
            this.result = event.result;
            this.imgUrl = event.param.getSourceUrl();
            this.showMenu = true;
            this.offsetX = 0;
            this.offsetY = Math.max(this.uiContext!.px2lpx(event?.param.y() ?? 0) - 0, 0) - 300;
            return true;
          }
          return false;
        })
        .expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.TOP, SafeAreaEdge.BOTTOM])
        .bindPopup(this.showMenu,
          {
            builder: this.customMenu(),
            enableArrow: false,
            placement: Placement.LeftTop,
            offset: { x: this.offsetX, y: this.offsetY },
            mask: false,
            onStateChange: (e) => {
              if (!e.isVisible) {
                this.showMenu = false;
                this.result!.closeContextMenu();
              }
            }
          });
    }
    .height('100%')
    .width('100%');
  }
}
```

### The download pipeline

`PressToSaveOrCopy.zip#PressToSaveOrCopy/entry/src/main/ets/pages/Index.ets`

```ts
this.showMenu = false;
let imgSavedPath = '';
// 下载图片
try {
  this.delegate.onBeforeDownload((webDownloadItem: webview.WebDownloadItem) => {
    Logger.info('will start a download.');
    // 传入一个下载路径，并开始下载。
    imgSavedPath = this.uiContext.getHostContext()?.getApplicationContext().tempDir + '/' +
    webDownloadItem.getSuggestedFileName();
    webDownloadItem.start(imgSavedPath);
  });
  // 下载中
  this.delegate.onDownloadUpdated((webDownloadItem: webview.WebDownloadItem) => {
    Logger.info(`download update guid: ${webDownloadItem.getGuid()}`);
    Logger.info(`download update percent complete: ${webDownloadItem.getPercentComplete()}`);
    Logger.info(`download update speed: ${webDownloadItem.getCurrentSpeed()}`);
  });
  // 下载失败
  this.delegate.onDownloadFailed((webDownloadItem: webview.WebDownloadItem) => {
    Logger.info(`download failed guid: ${webDownloadItem.getGuid()}`);          // FIX (HW-19-0088)
    Logger.info(`download failed last error code: ${webDownloadItem.getLastErrorCode()}`);
  });
  // 下载完成
  this.delegate.onDownloadFinish((webDownloadItem: webview.WebDownloadItem) => {
    let msg: string = `download finish guid: ${webDownloadItem.getGuid()}`;
    Logger.info(msg);
    let context: UIContext = this.getUIContext();
    // 保存图片
    savePhotoToGallery(context, imgSavedPath);
    this.imgUrl = '';
  });
  // 开始下载
  this.webVC.setDownloadDelegate(this.delegate);
  this.webVC.startDownload(this.imgUrl);
} catch (error) {
  Logger.error(`ErrorCode: ${(error as BusinessError).code},  Message: ${(error as BusinessError).message}`);
}
```

### Saving into the gallery (as shipped - see HW-19-0087)

`PressToSaveOrCopy.zip#PressToSaveOrCopy/entry/src/main/ets/pages/Index.ets`

```ts
async function savePhotoToGallery(uiContext: UIContext, imgPath: string) {
  let context = uiContext.getHostContext() as common.UIAbilityContext;
  let phAccessHelper = photoAccessHelper.getPhotoAccessHelper(context);
  try {
    // 指定待保存到媒体库的位于应用沙箱的图片uri。
    let srcFileUris: Array<string> = [
      imgPath                          // FIX: fileUri.getUriFromPath(imgPath)
    ];
    // 指定待保存照片的创建选项，包括文件后缀和照片类型，标题和照片子类型可选。
    let photoCreationConfigs: Array<photoAccessHelper.PhotoCreationConfig> = [
      {
        fileNameExtension: 'jpg',
        photoType: photoAccessHelper.PhotoType.IMAGE,
        subtype: photoAccessHelper.PhotoSubtype.DEFAULT
      }
    ];
    // 基于弹窗授权的方式获取媒体库的目标uri。
    let desFileUris: Array<string> = await phAccessHelper.showAssetsCreationDialog(srcFileUris, photoCreationConfigs);
    // 将来源于应用沙箱的照片内容写入媒体库的目标uri。
    let desFile: fileIo.File = await fileIo.open(desFileUris[0], fileIo.OpenMode.WRITE_ONLY);
    let srcFile: fileIo.File = await fileIo.open(imgPath, fileIo.OpenMode.READ_ONLY);
    await fileIo.copyFile(srcFile.fd, desFile.fd);
    fileIo.closeSync(srcFile);
    fileIo.closeSync(desFile);
    Logger.info('create asset by dialog successfully');
    showToast(uiContext, $r('app.string.save_success'));
  } catch (err) {
    Logger.error(`failed to create asset by dialog successfully errCode is: ${err.code}, ${err.message}`);
    showToast(uiContext, $r('app.string.save_failure'));
  }
}
```

### Copying the link

`PressToSaveOrCopy.zip#PressToSaveOrCopy/entry/src/main/ets/pages/Index.ets`

```ts
Text($r('app.string.copy_button_name'))
  .onClick(() => {
    this.showMenu = false;
    let pasteboardData = pasteboard.createData(pasteboard.MIMETYPE_TEXT_PLAIN, this.imgUrl);
    pasteboard.getSystemPasteboard().setData(pasteboardData, (error) => {
      if (!error) {
        showToast(this.getUIContext(), $r('app.string.copy_success'));
      } else {
        showToast(this.getUIContext(), $r('app.string.copy_failure'));
      }
      this.imgUrl = '';
    });
  })
```

### The toast helper

`PressToSaveOrCopy.zip#PressToSaveOrCopy/entry/src/main/ets/pages/Index.ets`

```ts
function showToast(uiContext: UIContext, res: Resource) {
  let promptAction: PromptAction = uiContext?.getPromptAction();
  promptAction.showToast({
    message: res,
    textColor: '#0A59F7'
  });
}
```

## Permissions & config

`ohos.permission.INTERNET` is required and is the only permission the document
lists and the sample declares - the page's images and the download are all
remote https URLs.

`PressToSaveOrCopy.zip#PressToSaveOrCopy/entry/src/main/module.json5`:

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" }
]
```

**No media permission is needed**: `showAssetsCreationDialog` is the
authorisation dialog itself, and the user's confirmation is what grants the
write. The `module.json5` `label` and `icon` under `abilities` do matter here -
the guide notes the dialog uses them to display the application name, and without
them no name is shown.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. `onContextMenuShow` dates from
  API 9.
- **`onContextMenuShow` must return `true`** for the application menu to replace
  the built-in one; returning `false` (as this handler does for non-image
  targets) restores the default.
- **`closeContextMenu()` is mandatory.** The web engine holds context-menu state
  until it is called, which is why it lives in `onStateChange` rather than in the
  menu buttons.
- **`onBeforeDownload` must call `start(path)`** - the download is queued, not
  started, until then.
- **The download destination is `tempDir`**, which the system may reclaim; the
  file only needs to survive until `showAssetsCreationDialog` has copied it.
- **`showAssetsCreationDialog` wants a URI.** A sandbox path saves but cannot be
  previewed in the dialog.
- **`fileAccess` defaults to false since API 12**, and it never affects
  `$rawfile` loading.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **Passing the sandbox path to `showAssetsCreationDialog` is incorrect.** The
  guide asks for a URI and states that a sandbox path can be saved "but cannot be
  previewed" - so the user approves a dialog that shows nothing. Convert with
  `fileUri.getUriFromPath`. (HW-19-0087)
- **`onDownloadFailed` only logs at info level, which is incorrect.** The save
  silently does nothing, and the failure line sits among the routine progress
  logs. The `save_failure` string resource already exists - use it.
  (HW-19-0088)
- **`.fileAccess(true)` grants capability the page does not need.** All images are
  remote and `$rawfile` loading is unaffected by the setting; the sibling H5
  samples in this industry set `false`. (HW-19-0089)
- **`this.result!.closeContextMenu()` uses a non-null assertion on a field that
  starts `undefined`.** It is safe only because the popup can only become visible
  from `onContextMenuShow`; a guard costs nothing and removes the assumption.
- **`offsetY = Math.max(px2lpx(y) - 0, 0) - 300`** contains a no-op `- 0` and a
  magic `- 300` that hardcodes the menu height. Derive the offset from the menu's
  measured height instead.
- **One `WebDownloadDelegate` is reused and re-registered on every save tap.**
  Each tap re-installs all four callbacks with a fresh `imgSavedPath` closure;
  register once in `aboutToAppear` and keep the path in a field if you want the
  flow to be readable.
- **`fileIo.closeSync` runs only on the success path.** A failing `copyFile`
  leaves both descriptors open - the official example has the same shape, so it
  is worth fixing deliberately with a `finally`.

## References

- `documentation/harmonyos-guides/05_media/photoaccesshelper-savebutton.md` - the
  save-by-dialog flow: "Specify the URI of the application file to be saved",
  the `file://` example, the `label`/`icon` dependency, and "If the passed URI is
  a sandbox path, images or videos can be saved but cannot be previewed."
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-savebutton
- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-events.md` -
  `onContextMenuShow`, `WebContextMenuParam` and `WebContextMenuResult`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-basic-components-web-events#oncontextmenushow9
- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-attributes.md` -
  `fileAccess`, its API 12 default and the `$rawfile` exemption.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-basic-components-web-attributes
- `documentation/harmonyos-guides/03_application-framework/web-menu.md` - using
  the Web component menu to handle page content.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/web-menu
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-attributes-popup#bindpopup -
  `bindPopup`, `Placement`, `offset`, `onStateChange`.
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `open`, `copyFile`, `closeSync`, `OpenMode`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/03_system/js-apis-pasteboard.md` -
  `createData`, `MIMETYPE_TEXT_PLAIN`, `getSystemPasteboard().setData`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-pasteboard
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/press_to_save_or_copy-0000002328179937
