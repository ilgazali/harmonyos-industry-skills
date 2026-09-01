---
id: OFFICE-08
title: File download and preview - request.downloadFile into the sandbox, then Preview Kit by MIME type
industry: 05_office
doc: huawei_industry_tree/05_office/docs/08_file_view.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/file_view-0000002256155688
sample: huawei_industry_tree/05_office/downloads/FileView.zip
kits: ["@kit.PreviewKit", "@kit.BasicServicesKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: ["request.downloadFile", "DownloadTask.on('progress')", "DownloadTask.on('complete')", "DownloadTask.on('fail')", "DownloadTask.off('progress')", "DownloadTask.getTaskInfo", "filePreview.canPreview", "filePreview.openPreview", "filePreview.PreviewInfo", "fileUri.getUriFromPath", "fs.accessSync", "fs.openSync", "fs.writeSync", "fs.close", "fs.closeSync", "resourceManager.getRawFileContent", "resourceManager.getStringSync", "systemDateTime.getTime", "UIContext.getHostContext", "UIContext.getPromptAction", "UIContext.px2vp", "@StorageProp"]
permissions: ["ohos.permission.INTERNET"]
min_api: 20
modules: [entry]
findings: [HW-05-0045, HW-05-0046, HW-05-0047, HW-05-0048, HW-05-0049, HW-05-0050, HW-05-0186]
status: verified-with-fixes
---

## When to use

Load this card when an office app has to **hand a user a document it does not
render itself** - a contract, a spreadsheet, an image attachment - and the
requirement is "open it", not "display it inline".

The pattern is three steps and no custom viewer:

1. get the bytes into the **application sandbox** (download over the network, or
   copy out of `rawfile` for bundled samples),
2. turn the sandbox path into a URI and ask **`filePreview.canPreview`** whether
   the system can show it,
3. hand a `PreviewInfo` with the correct **MIME type** to
   `filePreview.openPreview`.

Preview Kit renders the file through WPS Office, so the app needs no PDF, Word
or Excel engine of its own - and no storage permission, because everything
stays inside `context.cacheDir`.

Note the region and device limits under Constraints before choosing this route.

## Feature checklist

The application must be able to:

- List the offered document types and open one on tap.
- Reuse an already-cached copy instead of downloading again.
- Download a remote file into `context.cacheDir` with `request.downloadFile`.
- Report download progress once, report a download failure, and release the
  task's listeners.
- Copy bundled `rawfile` assets into the sandbox and wait for that copy to finish
  before previewing.
- Derive the MIME type from the file-name extension and refuse unknown types with
  a message.
- Check `canPreview` before `openPreview`, and tell the user when a file cannot be
  previewed.
- Handle rejections from both preview calls.

## Architecture

Single `entry` HAP:

| File | Responsibility |
| --- | --- |
| `pages/Index.ets` | `@Entry`; header, insets from `AppStorage`, and the `aboutToAppear` pre-copy of the three bundled rawfiles |
| `views/FileMainView.ets` | the document list; each row calls one `viewXxx` entry point |
| `utils/FileViewUtil.ets` | the whole feature: `myRawfileCopy`, `downloadFile`, `viewFile`, and the per-type entry points `viewPDF` / `viewTXT` / `viewJPG` / `viewWord` / `viewExcel` |
| `common/Constants.ets` | a two-entry `Record<string, number>` |
| `entryability/`, `entrybackupability/` | ability entry and backup extension |

Two sources feed the sandbox:

- **Network** - `viewPDF` and `viewJPG` read a URL from a string resource
  (`pdf_url`, `jpg_url`), build a unique file name from
  `systemDateTime.getTime()`, and go through `downloadFile`.
- **Bundled rawfile** - `viewTXT`, `viewWord` and `viewExcel` copy
  `file.txt` / `file.docx` / `file.xlsx` out of `resources/rawfile` with
  `myRawfileCopy`. `Index.aboutToAppear` also pre-copies all three at startup.

Both converge on `viewFile(fileName, filePath, uiContext)`, which is the only
place that touches Preview Kit.

Preview flow:

```
viewXxx(uiContext)
  -> (network)  downloadFile(url, fileName, uiContext)
                  fs.accessSync(cacheDir/fileName) ?
                    yes -> viewFile(...)
                    no  -> request.downloadFile(context, { url, filePath, enableMetered: true })
                             on('progress') / on('complete') -> getTaskInfo() -> viewFile(...)
  -> (rawfile)  myRawfileCopy(fileName, context)   [asynchronous]
                viewFile(...)                      [runs immediately - see HW-05-0045]

viewFile
  -> fileUri.getUriFromPath(filePath)
  -> filePreview.canPreview(context, uri)
       true  -> extension -> MIME switch -> PreviewInfo{title, uri, mimeType}
             -> filePreview.openPreview(context, fileInfo)
       false -> toast 'cannot preview'
```

## Implementation steps

1. **Declare `ohos.permission.INTERNET`** - that is the only permission this
   scenario needs. The sandbox is the app's own `cacheDir` and Preview Kit takes
   the URI it is given, so no storage or media permission is involved.
2. **Cache first.** In `downloadFile`, build `filePath = context.cacheDir + '/' +
   fileName` and check `fs.accessSync(filePath)` inside a `try/catch`; when the
   file is already there, go straight to `viewFile`.
3. **Download with `enableMetered: true`** so the transfer is allowed on a
   metered connection, and attach `.catch()` to the `downloadFile` promise - the
   sample already does this and surfaces `err.message` in a toast.
4. **Subscribe correctly.** Register `on('complete')` to trigger the preview,
   `on('fail')` to report a failed transfer, and use `on('progress')` only to
   drive a progress indicator - not to raise a toast per event. Release all three
   with `off(...)` (HW-05-0048).
5. **Copy rawfiles asynchronously and wait.** `resourceManager.getRawFileContent`
   delivers its bytes through a callback, so `myRawfileCopy` must return a
   promise the caller awaits before previewing (HW-05-0045). Open the target with
   `CREATE | READ_WRITE | TRUNC`, write, and close **once** in a guarded `finally`
   using the synchronous `closeSync` (HW-05-0046).
6. **Convert the path.** `fileUri.getUriFromPath(filePath)` - Preview Kit takes a
   URI, not a sandbox path.
7. **Gate on `canPreview`.** Handle all three outcomes: `true` (continue),
   `false` (tell the user the type is not previewable) and rejection (log). The
   sample handles all three correctly.
8. **Map the extension to a MIME type.** Split the file name on `.`, take the
   last segment, and switch. Use the OOXML type for `.xlsx`, not the legacy
   `.xls` type (HW-05-0047). Fall through to a `default` branch that reports the
   unsupported type instead of guessing.
9. **Open the preview.** `filePreview.openPreview(context, { title, uri, mimeType })`
   with `.then()` and `.catch()`. If the type cannot be determined, the reference
   allows `mimeType: ''` and the system infers from the URI suffix.

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Cached-or-download, then preview

`FileView.zip#entry/src/main/ets/utils/FileViewUtil.ets`

```ts
export function downloadFile(url: string, fileName: string, uiContext: UIContext) {
  let context = uiContext.getHostContext() as common.UIAbilityContext;
  let cacheDir = context.cacheDir;
  let promptAction = uiContext.getPromptAction();
  try {
    let filePath = cacheDir + '/' + fileName;
    //判断是否已下载过文件
    let res = fs.accessSync(filePath);
    if (res) {
      viewFile(fileName, filePath, uiContext);
    } else {
      request.downloadFile(context, {
        url: url,
        filePath: filePath,
        enableMetered: true
      }).then((downloadTask: request.DownloadTask) => {
        let progressCallback = (receivedSize: number, totalSize: number) => {
          promptAction.showToast({ message: $r('app.string.utils_third') }); // per-tick toast - HW-05-0048
          hilog.info(0x0000, TAG, 'download receivedSize: %s, totalSize: %s', receivedSize, totalSize);
        };
        downloadTask.on('progress', progressCallback);
        downloadTask.on('complete', async () => {
          const taskInfo = await downloadTask.getTaskInfo();
          hilog.info(0x0000, TAG, 'downloadTask complete,status: %s', taskInfo.status);
          viewFile(fileName, filePath, uiContext);
        });
      }).catch((err: BusinessError) => {
        promptAction.showToast({ message: err.message });
        hilog.info(0x0000, TAG, 'Invoke downloadTask failed-----, message is: %s', err.message);
      });
    }
  } catch (error) {
    let err: BusinessError = error as BusinessError;
    hilog.error(0x0000, TAG, 'Invoke downloadTask failed-----, message is: %s', err.message);
  }
}
```

Corrected listener handling:

```ts
const progressCallback = (receivedSize: number, totalSize: number) => {
  this.progress = totalSize > 0 ? receivedSize / totalSize : 0;   // drive an indicator, not a toast
};
const completeCallback = async () => {
  downloadTask.off('progress', progressCallback);
  downloadTask.off('complete', completeCallback);
  downloadTask.off('fail', failCallback);
  viewFile(fileName, filePath, uiContext);
};
const failCallback = (err: number) => {
  downloadTask.off('progress', progressCallback);
  downloadTask.off('complete', completeCallback);
  downloadTask.off('fail', failCallback);
  promptAction.showToast({ message: $r('app.string.utils_sec') });
  hilog.error(0x0000, TAG, 'download failed, code %{public}d', err);
};
downloadTask.on('progress', progressCallback);
downloadTask.on('complete', completeCallback);
downloadTask.on('fail', failCallback);
```

### Extension to MIME type, then preview

`FileView.zip#entry/src/main/ets/utils/FileViewUtil.ets`

```ts
export function viewFile(fileName: string, filePath: string, uiContext: UIContext) {
  let context = uiContext.getHostContext() as common.UIAbilityContext;
  let promptAction = uiContext.getPromptAction();
  // 将沙箱路径转换为uri
  let uri = fileUri.getUriFromPath(filePath);
  filePreview.canPreview(context, uri).then((result) => {
    if (result === true) {
      let fileNames: string[] = fileName.split('.');
      let type = fileNames[fileNames.length - 1];
      let mType: string = '';
      switch (type) {
        case 'pdf':  { mType = 'application/pdf'; break; }
        case 'txt':  { mType = 'text/plain'; break; }
        case 'png':  { mType = 'image/png'; break; }
        case 'jpg':  { mType = 'image/jpeg'; break; }
        case 'docx': { mType = 'application/vnd.openxmlformats-officedocument.wordprocessingml.document'; break; }
        case 'xlsx': { mType = 'application/vnd.ms-excel'; break; }   // wrong type - HW-05-0047
        default: {
          hilog.error(0x0000, TAG, 'Type Is Not Support');
          promptAction.showToast({ message: $r('app.string.utils_first') });
          return;
        }
      }
      let fileInfo: filePreview.PreviewInfo = {
        title: fileName,
        uri: uri,
        mimeType: mType
      };
      filePreview.openPreview(context, fileInfo).then(() => {
        hilog.info(0x0000, TAG, 'openPreview success');
      }).catch((err: BusinessError) => {
        hilog.error(0x0000, TAG, 'openPreview failed, err = %s', err.message);
      });
    } else {
      promptAction.showToast({ message: $r('app.string.utils_sec') });
      hilog.info(0x0000, TAG, 'openPreview failed, err = canPreview false');
    }
  }).catch((err: BusinessError) => {
    hilog.error(0x0000, TAG, 'canPreview failed, err = %s', err.message);
  });
}
```

Corrected `.xlsx` branch:

```ts
case 'xlsx': { mType = 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'; break; }
```

This `viewFile` is the part of the sample worth copying verbatim: it handles the
`canPreview` false branch, both rejections, and unknown extensions.

### Rawfile copy into the sandbox

`FileView.zip#entry/src/main/ets/utils/FileViewUtil.ets`

```ts
// as shipped - see HW-05-0046
export function myRawfileCopy(filetype: string, context: common.UIAbilityContext) {
  context.resourceManager.getRawFileContent(filetype, (err: BusinessError, data: Uint8Array) => {
    if (err != null) {
      hilog.info(0x0000, TAG, 'open file failed: ' + err.message);
    } else {
      let buffer = data.buffer;
      let file: fs.File = null!;
      try {
        file = fs.openSync(context.cacheDir + '/' + filetype, fs.OpenMode.CREATE | fs.OpenMode.READ_WRITE);
        fs.writeSync(file.fd, buffer);
      } catch (err) {
        hilog.info(0x0000, TAG, 'myRawfileCopy error');
      } finally {
        fs.close(file.fd);       // async, unawaited, and throws when openSync failed
      }
    }
  });
}
```

Corrected, awaitable form:

```ts
export async function myRawfileCopy(filetype: string, context: common.UIAbilityContext): Promise<void> {
  const data = await context.resourceManager.getRawFileContent(filetype);
  let file: fs.File | undefined = undefined;
  try {
    file = fs.openSync(context.cacheDir + '/' + filetype,
      fs.OpenMode.CREATE | fs.OpenMode.READ_WRITE | fs.OpenMode.TRUNC);
    fs.writeSync(file.fd, data.buffer);
  } catch (error) {
    hilog.error(0x0000, TAG, 'myRawfileCopy failed: %{public}s', JSON.stringify(error));
    throw error as Error;
  } finally {
    if (file) {
      fs.closeSync(file);
    }
  }
}

// callers then await the copy before previewing
export async function viewTXT(uiContext: UIContext) {
  const context = uiContext.getHostContext() as common.UIAbilityContext;
  const fileName = 'file.txt';
  const filePath = context.cacheDir + '/' + fileName;
  await myRawfileCopy(fileName, context);
  viewFile(fileName, filePath, uiContext);
}
```

### Per-type entry points

`FileView.zip#entry/src/main/ets/utils/FileViewUtil.ets`

```ts
export function viewPDF(uiContext: UIContext) {
  let context = uiContext.getHostContext() as common.UIAbilityContext;
  try {
    let pdfUrl: string = context.resourceManager.getStringSync($r('app.string.pdf_url').id);
    let time = systemDateTime.getTime();
    let fileName = 'PDF' + time + '文件.pdf';
    downloadFile(pdfUrl, fileName, uiContext);
  } catch (error) {
    let err: BusinessError = error as BusinessError;
    hilog.error(0x0000, TAG, `Invoke downloadTask failed-----, code is ${err.code}, message is ${err.message}`);
  }
}
```

`systemDateTime.getTime()` in the file name is what makes each download land on a
fresh path, so the `fs.accessSync` cache check never short-circuits a network
document - only re-taps within the same millisecond would.

### Startup pre-copy

`FileView.zip#entry/src/main/ets/pages/Index.ets`

```ts
@StorageProp('topRectHeight') topRectHeight: number = CONFIGURATION.FILE_VIEW_ZERO;
@StorageProp('bottomRectHeight') bottomRectHeight: number = CONFIGURATION.FILE_VIEW_ZERO;
uiContext = this.getUIContext();
context = this.uiContext.getHostContext();

aboutToAppear(): void {
  this.uiContext = this.getUIContext();
  this.context = this.uiContext.getHostContext();
  myRawfileCopy('file.txt', this.context as common.UIAbilityContext);
  myRawfileCopy('file.docx', this.context as common.UIAbilityContext);
  myRawfileCopy('file.xlsx', this.context as common.UIAbilityContext);
}
```

## Permissions & config

`FileView.zip#entry/src/main/module.json5`

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "pages": "$profile:main_pages",
    "requestPermissions": [
      {
        "name": "ohos.permission.INTERNET",
        "reason": "$string:EntryAbility_desc",
        "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
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

- `ohos.permission.INTERNET` is the only permission and it matches the document's
  权限说明 section exactly - verified consistent.
- **No storage or media permission.** Downloads land in `context.cacheDir`, and
  Preview Kit consumes the URI the app hands it.
- The `EntryBackupAbility` extension and its `backup_config` profile are part of
  the shipped module but missing from the document's project tree (HW-05-0050).
- Bundled assets live in `entry/src/main/resources/rawfile/`: `file.txt`,
  `file.docx`, `file.xlsx`. The remote URLs are `https://` string resources
  (`pdf_url`, `jpg_url`).
- `build-profile.json5` pins `compatibleSdkVersion` / `targetSdkVersion` to
  `6.0.0(20)`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **Region.** The Preview Kit guide states: "Currently, Preview Kit is available
  only in the Chinese mainland." Plan a fallback for any other market.
- **Devices.** "Preview Kit is supported on Huawei phones, tablets, and 2-in-1
  devices" - narrower than the module's own `deviceTypes` list would suggest for
  other features.
- **Emulator.** "The Emulator does not support preview of document files in
  formats such as .pdf, .pptx, .xlsx, and .docx." Test the document formats on a
  real device.
- **Rendering is delegated.** "Currently, Preview Kit uses WPS Office to implement
  preview capabilities. The preview screen displays 'Powered by WPS Office'."
  The app does not control the preview chrome.
- **Supported types are fixed.** The guide's table lists the extensions and MIME
  types Preview Kit accepts; `canPreview` is the runtime check, and the sample's
  `default` branch is the right response to anything outside the table.
- **`canPreview` is a weak check.** Per the API reference it "checks only whether
  the file exists and whether the file format is supported", so a `true` result
  does not guarantee `openPreview` succeeds - which is why the sample keeps a
  `.catch()` on `openPreview`.
- **Cache directory.** Files land in `context.cacheDir`, which the system may
  reclaim; the `fs.accessSync` check before each download is what makes that safe.

## Pitfalls

- **Calling `viewFile` on the line after `myRawfileCopy` is incorrect.** The copy
  runs inside a `getRawFileContent` callback, so the preview races an
  incompletely written file. Make the copy awaitable and await it. (HW-05-0045)
- **`finally { fs.close(file.fd); }` with `file` initialised to `null!` is
  incorrect.** When `openSync` throws, the finally raises a TypeError that hides
  the real failure; and `fs.close` is a promise that is neither awaited nor
  caught. Use `if (file) { fs.closeSync(file); }`. (HW-05-0046)
- **Mapping `.xlsx` to `application/vnd.ms-excel` is incorrect.** That is the
  `.xls` type; the official Preview Kit table maps `.xlsx` to
  `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`. The `.docx`
  branch in the same switch already follows the OOXML convention. (HW-05-0047)
- **Raising a toast inside the `progress` callback is incorrect** - it fires
  repeatedly during a transfer. Also register `on('fail')`, without which a failed
  download never calls back at all, and release every listener with `off(...)`.
  (HW-05-0048)
- **The scenario description naming png is incorrect**: the list offers `.jpg`,
  and the `.txt` row appears twice. The downloaded image file is even named
  `'png' + time + '文件.jpg'`. (HW-05-0049)
- **The project tree omitting `entrybackupability` is incorrect** - the directory
  ships in the sample and is declared as an `extensionAbility` in `module.json5`;
  the sibling document `07_camera_page.md` lists the same directory. (HW-05-0050)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-guides/07_application-services/preview-introduction.md` -
  the supported extension/MIME table, the Chinese-mainland region limit, the
  phone/tablet/2-in-1 device list, the emulator restriction and the WPS Office
  note.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/preview-introduction
- `documentation/harmonyos-references/03_application-services/preview-arkts.md` -
  `canPreview` / `openPreview` signatures, `PreviewInfo.mimeType` (including the
  empty-string inference rule) and the "checks only whether the file exists"
  caveat.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/preview-arkts
- `documentation/harmonyos-references/03_system/js-apis-request.md` -
  `request.downloadFile`, `on('progress')`, `on('complete')`, `on('fail')` and the
  `off(...)` counterparts.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-request
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `fs.close` (promise) versus `fs.closeSync`, `openSync` modes, and `accessSync`
  throwing rather than returning false.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
