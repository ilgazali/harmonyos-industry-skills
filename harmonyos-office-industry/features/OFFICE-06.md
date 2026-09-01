---
id: OFFICE-06
title: Document approval - canvas signature pad plus download, preview and save-as of the approval file
industry: 05_office
doc: huawei_industry_tree/05_office/docs/06_document_approval.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/document_approval-0000002280673593
sample: huawei_industry_tree/05_office/downloads/DocumentApproval.zip
kits: ["@kit.ArkUI", "@kit.CoreFileKit", "@kit.PreviewKit", "@kit.BasicServicesKit", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: [Canvas, CanvasRenderingContext2D, RenderingContextSettings, Path2D, "Path2D.moveTo", "Path2D.lineTo", "CanvasRenderingContext2D.stroke", "CanvasRenderingContext2D.beginPath", "CanvasRenderingContext2D.clearRect", "CanvasRenderingContext2D.getPixelMap", "CanvasRenderingContext2D.globalCompositeOperation", "onTouch", TouchType, HitTestMode, PanGesture, Scroller, "request.downloadFile", "DownloadTask.on('complete')", "DownloadTask.on('fail')", "DownloadTask.off('complete')", "picker.DocumentViewPicker", "picker.DocumentSaveOptions", "DocumentViewPicker.save", "fileUri.getUriFromPath", "fileUri.FileUri", "filePreview.canPreview", "filePreview.openPreview", "filePreview.PreviewInfo", "fs.accessSync", "fs.stat", "fs.openSync", "fs.copyFile", "fs.closeSync", Navigation, NavPathStack, NavDestination, routerMap, "@ComponentV2", "@Local", "@Param", "@Provider", "@Consumer"]
permissions: ["ohos.permission.INTERNET"]
min_api: 20
modules: [entry]
findings: [HW-05-0030, HW-05-0031, HW-05-0032, HW-05-0033, HW-05-0034, HW-05-0035, HW-05-0036, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when an approval workflow needs the two things a paper process
had: **read the document, then sign it**.

Concretely:

- a **freehand signature pad** built on `Canvas` + `Path2D`, with undo,
  re-sign and confirm, whose result is handed back to the approval page as a
  `PixelMap` and stamped onto the approval timeline;
- a **document viewer** that lazily downloads the file into the app sandbox on
  first tap and then opens it with the system file previewer;
- a **save-as** action that lets the user pick a location with
  `DocumentViewPicker` and copies the sandbox file there for permanent local
  storage.

The signature is drawn on a rotated canvas so the user signs in landscape while
the page stays portrait.

## Feature checklist

The application must be able to:

- Show the approval document rows, the process timeline and the approve/reject
  buttons.
- On first tap, download the remote file into `context.cacheDir`; on later taps
  reuse the cached copy.
- Report a download failure to the user rather than silently doing nothing.
- Check that the downloaded file is previewable, then open it in the system
  previewer with an explicit `mimeType`.
- Offer a download icon that opens `DocumentViewPicker.save()` with a proposed
  file name and copies the sandbox file into the chosen location.
- Provide a signature canvas that records one `Path2D` per stroke.
- Undo the last stroke by clearing the canvas and re-stroking the remaining
  paths.
- Clear the whole signature, and confirm it as a `PixelMap` returned to the
  approval page.
- Let a two-finger pan scroll the oversized signature canvas without disturbing
  the single-finger drawing gesture.

## Architecture

Single `entry` HAP. The whole feature is built on **ArkUI state management V2**
(`@ComponentV2`, `@Local`, `@Param`, `@Provider`, `@Consumer`), not V1.

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | full-screen layout, loads `pages/ApprovalPage` |
| `entrybackupability/EntryBackupAbility.ets` | backup extension stub |
| `pages/ApprovalPage.ets` | `@Entry @ComponentV2`; owns the `Navigation`/`NavPathStack`, the document rows, the timeline, and `filePreview()`; `@Provider`s `context`, `fileName`, `filePath`, `url`, `pixelMap`, `navPathStack` |
| `pages/SignaturePage.ets` | the signature `NavDestination`, reached by route name `signaturePage`; `@Consumer`s `pixelMap` and `navPathStack` |
| `component/FileComponent.ets` | one document row plus its download icon; `@Consumer`s the shared file state, `@Param`s its own title/icon/size |
| `component/CommonTimeLine.ets` | the approval-process timeline item |
| `utils/FileUtil.ets` | `viewFile` (canPreview + openPreview) and `fileDownload` / `writeToFile` (picker + copy) |
| `utils/Logger.ets`, `common/Constants.ets` | logging wrapper, `FILE_NAME` |

The parent-to-child channel is `@Provider` / `@Consumer` by name: `ApprovalPage`
provides `filePath`, `fileName`, `url` and `context`, and both `FileComponent`
and `SignaturePage` consume them without any explicit parameter passing.
`pixelMap` travels the other way - `SignaturePage` writes the confirmed
signature into the consumed `pixelMap`, and the provider in `ApprovalPage`
re-renders the stamp.

Navigation is declared in `route_map.json` (`signaturePage` ->
`signaturePageBuilder` in `SignaturePage.ets`) and reached with
`navPathStack.pushPathByName('signaturePage', null)`.

Preview flow:

```
document row onClick -> ApprovalPage.filePreview()
  fs.accessSync(filePath) ?
    yes -> fs.stat(filePath) -> size > 0 -> viewFile(fileName, filePath, context)
    no  -> request.downloadFile(context, { url, filePath })
             -> downloadTask.on('complete') -> viewFile(...)
  viewFile -> fileUri.getUriFromPath(filePath)
           -> filePreview.canPreview(context, uri)
           -> filePreview.openPreview(context, { title, uri, mimeType: 'application/pdf' })
```

Save-as flow:

```
download icon onClick -> FileComponent
  fs.accessSync(filePath) ? yes -> isFile() : download first, then isFile()
  isFile() -> fs.stat -> size > 0 -> fileDownload(fileName, filePath, uiContext)
  fileDownload -> new picker.DocumentSaveOptions(); newFileNames = [fileName]
               -> new picker.DocumentViewPicker().save(options)
               -> writeToFile(documentSaveResult[0], filePath, uiContext)
  writeToFile -> new fileUri.FileUri(uri).path -> fs.openSync(dstPath, READ_WRITE|CREATE)
              -> fs.copyFile(filePath, file.fd) -> close
```

## Implementation steps

1. **Declare `ohos.permission.INTERNET`** in `module.json5` and add
   `"routerMap": "$profile:route_map"` with an entry binding `signaturePage` to
   the page's global `@Builder`.
2. **Provide the shared file state once.** In `ApprovalPage`, `@Provider()` the
   `context`, the `fileName`, the sandbox `filePath`
   (`context.cacheDir + '/' + fileName`) and the remote `url` read from a string
   resource. Children consume them by the same names.
3. **Download lazily into the cache directory.** Guard with `fs.accessSync`
   inside a `try/catch` (HW-05-0036); when the file is absent call
   `request.downloadFile(context, { url, filePath })`, attach `.catch()`
   (HW-05-0031), and register **both** `on('complete')` and `on('fail')`,
   releasing them with `off(...)` when the page goes away (HW-05-0032).
4. **Verify the cached file is non-empty.** `fs.stat(filePath)` and only proceed
   when `stat.size > 0` - a partially written cache file would otherwise be
   handed to the previewer.
5. **Preview through Preview Kit.** Convert the sandbox path with
   `fileUri.getUriFromPath`, call `filePreview.canPreview(context, uri)`, and on
   `true` call `filePreview.openPreview(context, { title, uri, mimeType })`.
   Attach `.catch()` to both and handle the `false` result (HW-05-0033). Passing
   an explicit `mimeType` is optional - the reference notes an empty string makes
   the system infer the type from the URI suffix.
6. **Save-as with the document picker.** Construct a
   `picker.DocumentSaveOptions`, set `newFileNames = [fileName]` so the dialog
   proposes a name, call `new picker.DocumentViewPicker().save(options)`, guard
   the returned array against being empty (the user cancelled), then copy.
7. **Copy correctly.** Resolve the picked URI with `new fileUri.FileUri(uri).path`,
   open it `READ_WRITE | CREATE`, and **await** `fs.copyFile(srcPath, file.fd)`
   before closing the descriptor in `finally` (HW-05-0030).
8. **Build the signature canvas.** One `CanvasRenderingContext2D` created from
   `new RenderingContextSettings(true)`; in `onReady` reset `pathArray` and set
   `strokeStyle` / `lineWidth`.
9. **One `Path2D` per stroke.** In `onTouch`, guard the optional event
   (`event?.touches.length === 1`), and on `TouchType.Down` call `beginPath()`,
   assign a **new** `Path2D`, push it into `pathArray` and `moveTo` the first
   point; on `TouchType.Move` `lineTo` and `stroke(this.tempPath)`
   (HW-05-0035).
10. **Undo by replay.** `clearRect` the whole canvas, `pathArray.pop()`, reset
    `globalCompositeOperation = 'source-over'`, then re-stroke every remaining
    path inside its own `beginPath()` / `closePath()` pair.
11. **Confirm as a PixelMap.** `this.pixelMap = this.canvasContext.getPixelMap(0,
    0, 700, 350)` writes through the `@Consumer` back to the provider, then
    `navPathStack.pop()`. The reference warns that `getPixelMap` "involves
    time-consuming memory copy", so call it once on confirm, never per frame.
12. **Separate the two gestures.** Put the canvas inside a `Scroll` with
    `enableScrollInteraction(false)` and drive scrolling from a
    `PanGesture({ fingers: 2 })` that calls `scroller.scrollTo`, so one finger
    always means "draw". Set `hitTestBehavior(HitTestMode.Transparent)` on the
    canvas.

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Signature strokes and undo

`DocumentApproval.zip#entry/src/main/ets/pages/SignaturePage.ets`

```ts
@Entry
@ComponentV2
struct SignaturePage {
  @Local paintSize: number = 10;
  @Local paintColor: Color = Color.Black;
  @Local pathArray: Array<Path2D | undefined> = [];
  @Consumer() pixelMap: PixelMap | undefined = undefined;
  @Consumer() navPathStack: NavPathStack = new NavPathStack();
  private settings: RenderingContextSettings = new RenderingContextSettings(true);
  canvasContext: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings);
  scroller: Scroller = new Scroller();
  tempPath: Path2D = new Path2D();

  // ...
  Canvas(this.canvasContext)
    .width(680)
    .height(290)
    .onReady(() => {
      this.pathArray = [];
      this.canvasContext.strokeStyle = this.paintColor;
      this.canvasContext.lineWidth = this.paintSize;
    })
    .onTouch((event?: TouchEvent) => {
      if (event?.touches.length === 1) {
        if (event.type === TouchType.Down) {
          this.canvasContext.beginPath();
          this.tempPath = new Path2D();
          this.pathArray.push(this.tempPath);
          this.tempPath.moveTo(event.touches[0].x, event.touches[0].y);
        }
        if (event.type === TouchType.Move) {
          this.tempPath.lineTo(event.touches[0].x, event.touches[0].y);
          this.canvasContext.stroke(this.tempPath);
        }
      }
    })
    .hitTestBehavior(HitTestMode.Transparent);
}
```

Undo, clear and confirm:

```ts
Button($r('app.string.revoke'))
  .onClick(() => {
    this.canvasContext.clearRect(0, 0, 700, 350);
    this.pathArray.pop();
    this.canvasContext.globalCompositeOperation = 'source-over';
    for (let i = 0; i < this.pathArray.length; i++) {
      this.canvasContext.beginPath();
      this.canvasContext.stroke(this.pathArray[i]);
      this.canvasContext.closePath();
    }
  });
Button($r('app.string.re_sign'))
  .onClick(() => {
    this.pathArray = [];
    this.canvasContext.clearRect(0, 0, 700, 720);
  });
Button($r('app.string.confirm'))
  .onClick(() => {
    this.pixelMap = this.canvasContext.getPixelMap(0, 0, 700, 350);
    this.navPathStack.pop();
  });
```

Two-finger scroll over a single-finger drawing surface:

```ts
Scroll(this.scroller) {
  Canvas(this.canvasContext) /* ... */
}
.enableScrollInteraction(false)
.gesture(
  PanGesture({ fingers: 2 })
    .onActionUpdate((event: GestureEvent) => {
      this.positionY = this.positionY - 0.3 * event.offsetY;
      this.scroller.scrollTo({ xOffset: 0, yOffset: this.positionY });
    })
);
```

### Shared file state and the lazy download

`DocumentApproval.zip#entry/src/main/ets/pages/ApprovalPage.ets`

```ts
@Entry
@ComponentV2
struct ApprovalPage {
  uiContext = this.getUIContext();
  @Provider() pixelMap: PixelMap | undefined = undefined;
  @Provider() context: Context = this.uiContext.getHostContext() as common.UIAbilityContext;
  @Provider() fileName: string = CommonConstant.FILE_NAME;      // '销售合同.pdf'
  cacheDir = this.context.cacheDir;
  @Provider() filePath: string = this.cacheDir + '/' + this.fileName;
  @Provider() url: string = this.uiContext.getHostContext()?.resourceManager.getStringByNameSync('file_url') as string;
  @Provider() navPathStack: NavPathStack = new NavPathStack();

  //预览文件，如之前未进行下载则先下载再预览。
  filePreview() {
    try {
      let res = fs.accessSync(this.filePath);
      if (res) {
        let fileSize = 0;
        fs.stat(this.filePath).then((stat: fs.Stat) => {
          fileSize = stat.size;
          if (fileSize > 0) {
            viewFile(this.fileName, this.filePath, this.context);
          }
        });
      } else {
        request.downloadFile(this.context, {
          url: this.url,
          filePath: this.filePath
        }).then((downloadTask: request.DownloadTask) => {
          downloadTask.on('complete', () => {          // no off / no on('fail') - HW-05-0032
            viewFile(this.fileName, this.filePath, this.context);
          });
        });                                            // no .catch - HW-05-0031
      }
    } catch (error) {
      let err: BusinessError = error as BusinessError;
      Logger.error(err.code + err.message);
    }
  }
}
```

Corrected download registration:

```ts
request.downloadFile(this.context, { url: this.url, filePath: this.filePath })
  .then((downloadTask: request.DownloadTask) => {
    this.downloadTask = downloadTask;
    this.completeCallback = () => viewFile(this.fileName, this.filePath, this.context);
    this.failCallback = (err: number) => Logger.error('testTag', `download failed, code ${err}`);
    downloadTask.on('complete', this.completeCallback);
    downloadTask.on('fail', this.failCallback);
  })
  .catch((err: BusinessError) => {
    Logger.error('testTag', `Failed to request the download. Code: ${err.code}, message: ${err.message}`);
  });

// and on teardown
aboutToDisappear(): void {
  this.downloadTask?.off('complete', this.completeCallback);
  this.downloadTask?.off('fail', this.failCallback);
}
```

### Preview through Preview Kit

`DocumentApproval.zip#entry/src/main/ets/utils/FileUtil.ets`

```ts
import { fileUri, picker } from '@kit.CoreFileKit';
import { filePreview } from '@kit.PreviewKit';

//文件预览
export function viewFile(fileName: string, filePath: string, context: Context) {
  // 将沙箱路径转换为uri
  let uri = fileUri.getUriFromPath(filePath);
  filePreview.canPreview(context, uri).then((result) => {
    if (result === true) {
      let fileInfo: filePreview.PreviewInfo = {
        title: fileName,
        uri: uri,
        mimeType: 'application/pdf'
      };
      filePreview.openPreview(context, fileInfo).then(() => {
      });                                    // no .catch on either - HW-05-0033
    }
  });
}
```

### Save-as through DocumentViewPicker

`DocumentApproval.zip#entry/src/main/ets/utils/FileUtil.ets`

```ts
//完成picker初始化
export function fileDownload(fileName: string, filePath: string, uiContext: UIContext) {
  let documentSaveOptions = new picker.DocumentSaveOptions();
  let documentPicker = new picker.DocumentViewPicker();
  documentSaveOptions.newFileNames = [fileName];
  documentPicker.save(documentSaveOptions).then((documentSaveResult: Array<string>) => {
    if (!documentSaveResult || documentSaveResult.length === 0) {
      return;
    }
    writeToFile(documentSaveResult[0], filePath, uiContext);
  }).catch((err: BusinessError) => {
    Logger.error('testTag', 'downloadAction.save failed with err: %{public}s', JSON.stringify(err) ?? '');
  });
}

//向本地创建的文件写入数据 - as shipped, see HW-05-0030
function writeToFile(uri: string, filePath: string, uiContext: UIContext) {
  if (!uri || uri === '') {
    return;
  }
  let uriString = new fileUri.FileUri(uri);
  let dstPath = uriString.path;
  let file = fs.openSync(dstPath, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
  try {
    fs.copyFile(filePath, file.fd).then(() => {
      uiContext.getPromptAction().showToast({ message: $r('app.string.download_suc') });
    });
  } catch (error) {
    uiContext.getPromptAction().showToast({ message: $r('app.string.download_fail') });
    Logger.error('testTag', '%{public}s', `write data to file failed with error message: ${error.message},
        error code: ${error.code}`);
  } finally {
    fs.closeSync(file);   // runs before the copy finishes
  }
}
```

Corrected copy:

```ts
async function writeToFile(uri: string, filePath: string, uiContext: UIContext) {
  if (!uri) {
    return;
  }
  const dstPath = new fileUri.FileUri(uri).path;
  const file = fs.openSync(dstPath, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
  try {
    await fs.copyFile(filePath, file.fd);
    uiContext.getPromptAction().showToast({ message: $r('app.string.download_suc') });
  } catch (error) {
    uiContext.getPromptAction().showToast({ message: $r('app.string.download_fail') });
    Logger.error('testTag', 'copy failed: %{public}s', JSON.stringify(error));
  } finally {
    fs.closeSync(file);
  }
}
```

### The document row

`DocumentApproval.zip#entry/src/main/ets/component/FileComponent.ets`

```ts
@ComponentV2
export struct FileComponent {
  uiContext = this.getUIContext();
  @Consumer() context: Context = this.uiContext.getHostContext() as common.UIAbilityContext;
  @Consumer() fileName: string = '';
  @Consumer() filePath: string = '';
  @Consumer() url: string = '';
  @Param title: Resource | string = '';
  @Param photo: Resource | string = '';
  @Param filesize: Resource | string = '';

  isFile() {
    let fileSize = 0;
    fs.stat(this.filePath).then((stat: fs.Stat) => {
      fileSize = stat.size;
      if (fileSize > 0) {
        fileDownload(this.fileName, this.filePath, this.uiContext);
      }
    });
  }
  // download icon onClick - see HW-05-0036 for the missing try/catch
}
```

### Route declaration

`DocumentApproval.zip#entry/src/main/resources/base/profile/route_map.json`

```json
{
  "routerMap": [
    {
      "name": "signaturePage",
      "pageSourceFile": "src/main/ets/pages/SignaturePage.ets",
      "buildFunction": "signaturePageBuilder",
      "data": {
        "description": "this is RecorderPage"
      }
    }
  ]
}
```

## Permissions & config

`DocumentApproval.zip#entry/src/main/module.json5`

```json5
{
  "module": {
    "routerMap": "$profile:route_map",
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "requestPermissions": [
      {
        "name": "ohos.permission.INTERNET",
        "reason": "$string:EntryAbility_desc",
        "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
      }
    ],
    "pages": "$profile:main_pages",
    "abilities": [
      {
        "name": "EntryAbility",
        "srcEntry": "./ets/entryability/EntryAbility.ets",
        "exported": true,
        "skills": [
          { "entities": ["entity.system.home"], "actions": ["action.system.home"] }
        ]
      }
    ],
    "extensionAbilities": [
      {
        "name": "EntryBackupAbility",
        "srcEntry": "./ets/entrybackupability/EntryBackupAbility.ets",
        "type": "backup",
        "exported": false,
        "metadata": [
          { "name": "ohos.extension.backup", "resource": "$profile:backup_config" }
        ]
      }
    ]
  }
}
```

Notes on the config:

- `ohos.permission.INTERNET` is the only permission, and it matches the
  document's 权限说明 section exactly - verified consistent.
- **No storage permission is needed.** The download target is the app's own
  `cacheDir`, and the save-as destination comes from `DocumentViewPicker`, which
  grants access to the single file the user picked.
- `routerMap` is required for `pushPathByName('signaturePage', ...)`; the
  `buildFunction` must name a global `@Builder` exported from the page file.
- The remote document URL lives in `resources/base/element/string.json` under
  `file_url` and is an `https://` address.
- `build-profile.json5` pins `compatibleSdkVersion` and `targetSdkVersion` to
  `6.0.0(20)` with
  `strictMode: { caseSensitiveCheck: true, useNormalizedOHMUrl: true }`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later; the sample pins `6.0.0(20)`.
- **Devices.** `deviceTypes` is `["phone", "tablet", "2in1"]`. `filePreview` is
  documented for Phone, PC/2in1 and Tablet - narrower than most window APIs, so
  the preview half of this scenario does not apply to TV or wearables.
- **State management V2 only.** Every struct here is `@ComponentV2` with
  `@Local` / `@Param` / `@Provider` / `@Consumer`. V1 decorators (`@State`,
  `@Provide`, `@Link`) cannot be mixed into these structs.
- **`canPreview` is a weak check.** The reference states it "checks only whether
  the file exists and whether the file format is supported. When viewing the file
  using openPreview, ensure that the authorization of the file can be
  transferred." A `true` result does not guarantee `openPreview` succeeds.
- **`getPixelMap` is expensive.** The reference warns it "involves time-consuming
  memory copy. Therefore, avoid frequent calls to it" - fine on a confirm button,
  not in a touch handler.
- **The signature geometry is hard-coded.** `clearRect(0, 0, 700, 350)`,
  `getPixelMap(0, 0, 700, 350)` and the re-sign `clearRect(0, 0, 700, 720)` use
  literal pixel bounds that do not match the declared canvas size (680 x 290);
  the canvas is also `.rotate({ angle: 90 })`. Re-derive these from the component
  size if you change the layout.
- **Cache directory, not files directory.** The download target is
  `context.cacheDir`, so the system may reclaim it; the `fs.accessSync` +
  `fs.stat` size check on every tap is what makes that safe.

## Pitfalls

- **Closing the destination file in `finally` around an un-awaited
  `fs.copyFile` is incorrect.** The copy is asynchronous, so the descriptor is
  closed while the write is still in flight; the success toast also fires
  unconditionally and the failure toast is unreachable. Await the copy inside the
  `try`. (HW-05-0030)
- **`request.downloadFile(...).then(...)` without `.catch()` is incorrect.** The
  official example attaches one; the surrounding synchronous `try/catch` cannot
  observe an asynchronous rejection, so a failed request produces no log and no
  UI change. (HW-05-0031)
- **Registering only `on('complete')` is incorrect.** A download that fails mid
  transfer never calls back at all - register `on('fail')` too - and the
  subscription is never released, so add the matching `off(...)`. (HW-05-0032)
- **`canPreview(...).then(...)` and `openPreview(...).then(() => {})` without
  `.catch()` are incorrect,** and the `result === false` branch is missing
  entirely; the reference examples attach a rejection handler to both.
  (HW-05-0033)
- **The document's download snippet does not compile, which is incorrect.**
  `documentViewPicker.save().then(()={ ... })` is not valid ArkTS, `save()` is
  missing its `DocumentSaveOptions`, and `documentSaveResult` is unbound. Use the
  shipped `fileDownload` implementation instead. (HW-05-0034)
- **The document's `onTouch` snippet is incorrect.** It reads `event.type` from a
  parameter declared `event?: TouchEvent`, and it pushes `this.tempPath` without
  first assigning a new `Path2D`, so every stroke shares one accumulating path and
  undo cannot work. Guard with `event?.touches.length === 1` and create a new
  `Path2D` on `TouchType.Down`. (HW-05-0035)
- **Calling `fs.accessSync` outside a `try/catch` in `FileComponent` is
  incorrect.** It throws - `13900012 Permission denied`, for example - rather than
  returning `false`, and the exception escapes the click handler so the download
  button does nothing. `ApprovalPage` already guards the identical call.
  (HW-05-0036)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/03_system/js-apis-request.md` -
  `request.downloadFile` with its `.catch()` example,
  `on('complete'|'pause'|'remove')`, `off('complete'|'pause'|'remove')` and
  `on('fail')`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-request
- `documentation/harmonyos-references/03_application-services/preview-arkts.md` -
  `canPreview` / `openPreview` signatures, the `PreviewInfo.mimeType` note, the
  "checks only whether the file exists" caveat and the `.catch()` examples.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/preview-arkts
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `fileIo.closeSync`, `accessSync` throwing `13900012`, and `copyFile`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/02_application-framework/ts-canvasrenderingcontext2d.md` -
  `getPixelMap(sx, sy, sw, sh)` and its "time-consuming memory copy" warning.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-canvasrenderingcontext2d
- `documentation/harmonyos-references/02_application-framework/js-apis-file-picker.md` -
  `DocumentViewPicker.save` and `DocumentSaveOptions.newFileNames`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-picker
