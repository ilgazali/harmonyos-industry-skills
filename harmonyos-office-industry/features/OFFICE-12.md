---
id: OFFICE-12
title: Electronic seal on a PDF - PDF Kit preview, addImageObject stamp, save through the document picker
industry: 05_office
doc: huawei_industry_tree/05_office/docs/12_pdf_add_mark.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/pdf_add_mark-0000002324100453
sample: huawei_industry_tree/05_office/downloads/PDFAddMark.zip
kits: ["@kit.PDFKit", "@kit.CoreFileKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [PdfView, "pdfViewManager.PdfController", "PdfController.loadDocument", "PdfController.releaseDocument", "PdfController.registerPageCountChangedListener", "pdfService.PdfDocument", "PdfDocument.loadDocument", "PdfDocument.getPage", "PdfDocument.saveDocument", "PdfDocument.releaseDocument", "pdfService.PdfPage", "PdfPage.addImageObject", "pdfService.ParseResult", "pdfService.PageFit", "picker.DocumentViewPicker", "DocumentViewPicker.save", "picker.DocumentSaveOptions", "fs.accessSync", "fs.openSync", "fs.writeSync", "fs.closeSync", "fileIo.copyFileSync", "resourceManager.getRawFileContentSync", "UIAbilityContext.terminateSelf", "UIContext.px2vp", "UIContext.getPromptAction", "@StorageProp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-05-0071, HW-05-0072, HW-05-0073, HW-05-0074, HW-05-0075, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when an office app has to **stamp a PDF with a company seal and
let the user keep the stamped copy** - the electronic equivalent of pressing a
chop onto a printed notice.

The scenario splits cleanly into three PDF Kit surfaces that are easy to
confuse:

| Surface | Type | Used for |
| --- | --- | --- |
| `PdfView` + `pdfViewManager.PdfController` | UI component + its controller | showing the document on screen |
| `pdfService.PdfDocument` | headless document model | loading, editing and saving the file |
| `DocumentViewPicker.save` | system picker | letting the user choose where the stamped copy goes |

The viewer and the editor are **separate objects over the same file**. Stamping
means: edit through `PdfDocument`, save to a new path, release the editor, then
tell the `PdfController` to reload from that new path.

No permission is needed anywhere in this flow.

## Feature checklist

The application must be able to:

- Copy the source PDF and the seal image out of `rawfile` into the sandbox on
  first launch, skipping the copy when they are already there.
- Render the PDF in a `PdfView` and report load success or failure.
- Stamp the seal image onto page 0 at a fixed position and size.
- Save the stamped result to a new timestamped sandbox path, leaving the original
  intact.
- Refuse a second stamp and say so.
- Reload the viewer from the stamped file **only when the save succeeded**.
- Let the user save the stamped PDF into their own documents through the system
  picker, with a proposed file name.
- Report a failure at every stage rather than reporting success unconditionally.

## Architecture

Single `entry` HAP, four source files:

| File | Responsibility |
| --- | --- |
| `pages/AddMarkPage.ets` | `@Entry`; the viewer, the stamp button, the save button, and `callFilePickerSaveFile` |
| `common/FileUtils.ets` | `copyResourceFile2Cache` (rawfile to sandbox) and `copyFile2Document` (sandbox to picked URI) |
| `common/Constants.ets` | the PDF and seal file names |
| `common/Logger.ets` | hilog wrapper |
| `entryability/`, `entrybackupability/` | ability entry and backup stub |

Two independent handles over the same document live side by side on the page:

```ts
private controller: pdfViewManager.PdfController = new pdfViewManager.PdfController(); // the viewer
private pdfDocument: pdfService.PdfDocument = new pdfService.PdfDocument();            // the editor
private filePath   = this.context.filesDir + '/' + Constants.PDF_NAME;   // the original
private outPdfPath = this.context.filesDir + '/' + Constants.PDF_NAME;   // becomes the stamped copy
```

`outPdfPath` starts equal to `filePath` and is reassigned to a timestamped name
the first time the user stamps, so the original is never overwritten.

Stamp flow:

```
aboutToAppear
  copyResourceFile2Cache(PDF_NAME)      rawfile -> filesDir
  copyResourceFile2Cache(MARKER_NAME)
  controller.registerPageCountChangedListener(...)
  await controller.loadDocument(filePath)          -> PdfView renders

mark button
  outPdfPath = filesDir + '/' + Date.now() + PDF_NAME
  pdfDocument.loadDocument(filePath)               -> ParseResult
  pdfDocument.getPage(0)
  pdfPage.addImageObject(imgPath, 370, 200, 120, 120)
  pdfDocument.saveDocument(outPdfPath)             -> boolean
  pdfDocument.releaseDocument()
  reloadDocument(outPdfPath)
     controller.releaseDocument()
     await controller.loadDocument(outPdfPath)

save button
  DocumentViewPicker.save({ newFileNames: [PDF_NAME] })
  -> uriSave = result[0]
  -> FileUtils.copyFile2Document(outPdfPath, uriSave)
```

The release/reload ordering matters: the editor must be released before the
viewer loads the new file, and the viewer must release its old document before
loading another.

## Implementation steps

1. **Declare no permission.** Reading `rawfile`, writing to `filesDir` and saving
   through `DocumentViewPicker` all work without one, and the sample declares
   none - matching the document, which has no 权限说明 section.
2. **Seed the sandbox once.** `fs.accessSync(filePath)` first; on a miss,
   `resourceManager.getRawFileContentSync('rawfile/' + fileName)` and write with
   `fs.OpenMode.WRITE_ONLY | CREATE | TRUNC`. Close in a guarded `finally`.
3. **Load the viewer.** Create a `pdfViewManager.PdfController`, register the
   page-count listener, and `await controller.loadDocument(filePath)`; compare the
   result against `pdfService.ParseResult.PARSE_SUCCESS`.
4. **Render.** `PdfView({ controller, pageFit, showScroll })` with an `.id()` so
   it can be addressed. The sample uses `PageFit.FIT_NONE`; the document's snippet
   says `FIT_WIDTH` (HW-05-0075).
5. **Stamp through a separate `PdfDocument`.** `loadDocument(filePath)`, check for
   `PARSE_SUCCESS`, `getPage(0)`, then
   `addImageObject(path, x, y, width, height)`.
6. **Save to a fresh path.** Build `filesDir + '/' + Date.now() + PDF_NAME` so the
   original survives, and **act on `saveDocument`'s boolean** rather than only
   logging it (HW-05-0071).
7. **Release and reload in the success branch only.**
   `pdfDocument.releaseDocument()` then
   `controller.releaseDocument()` + `await controller.loadDocument(outPdfPath)` -
   and never when the load or save failed, or the viewer is pointed at a file
   that does not exist (HW-05-0071).
8. **Guard the second stamp.** Keep an `isMarked` flag and toast instead of
   stamping twice.
9. **Save through the picker.** `new picker.DocumentSaveOptions()` with
   `newFileNames`, `new picker.DocumentViewPicker().save(options)`, then treat an
   empty result array as a cancellation and return (HW-05-0072).
10. **Copy defensively.** `copyFile2Document` must open in `try`, close in
    `finally`, and return `false` on failure - and the caller must branch on that
    boolean before showing a success toast (HW-05-0073, HW-05-0072).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Seeding the sandbox from rawfile

`PDFAddMark.zip#entry/src/main/ets/common/FileUtils.ets`

```ts
public static copyResourceFile2Cache(context: common.UIAbilityContext, fileName: string): boolean {
  let fdSand: fs.File = null!;
  try {
    let dir = context.filesDir;
    let filePath = dir + '/' + fileName;
    let res = fs.accessSync(filePath);
    if (res) {
      Logger.info(TAG, 'file exists');
    } else {
      let content = context.resourceManager.getRawFileContentSync('rawfile/' + fileName);
      fdSand = fs.openSync(filePath, fs.OpenMode.WRITE_ONLY | fs.OpenMode.CREATE | fs.OpenMode.TRUNC);
      fs.writeSync(fdSand.fd, content.buffer);
    }
    return true;
  } catch (e) {
    let err = e as BusinessError;
    Logger.error(TAG, `Get exception: ${err}`);
    return false;
  } finally {
    if (fdSand != null && fdSand.fd != null) {
      fs.closeSync(fdSand.fd);
    }
  }
}
```

This is the pattern to copy: `TRUNC` so a rewrite cannot leave a stale tail, a
`catch` that returns `false`, and a `finally` that closes only a handle that was
actually opened. Compare it with `copyFile2Document` below, which does none of
this (HW-05-0073).

### Viewer setup

`PDFAddMark.zip#entry/src/main/ets/pages/AddMarkPage.ets`

```ts
import { pdfService, PdfView, pdfViewManager } from '@kit.PDFKit';

private controller: pdfViewManager.PdfController = new pdfViewManager.PdfController();
private pdfDocument: pdfService.PdfDocument = new pdfService.PdfDocument();
private filePath = this.context.filesDir + '/' + Constants.PDF_NAME;
private outPdfPath: string = this.context.filesDir + '/' + Constants.PDF_NAME;

aboutToAppear(): void {
  (async () => {
    FileUtils.copyResourceFile2Cache(this.context, Constants.PDF_NAME);
    FileUtils.copyResourceFile2Cache(this.context, Constants.MARKER_NAME);
    this.controller.registerPageCountChangedListener((pageCount: number) => {
      Logger.info(TAG, 'registerPageCountChanged-', pageCount.toString());
    });
    let loadResult: pdfService.ParseResult = await this.controller.loadDocument(this.filePath);
    if (pdfService.ParseResult.PARSE_SUCCESS === loadResult) {
      Logger.info(TAG, 'aboutToAppear', 'PdfView Component has been successfully loaded!');
    }
  })();
}

// in build()
PdfView({
  controller: this.controller,
  pageFit: pdfService.PageFit.FIT_NONE,
  showScroll: false
})
  .id('pdfview_app_view')
  .backgroundColor(Color.White)
```

The `aboutToAppear` body is wrapped in an immediately-invoked async function
because the lifecycle hook itself cannot be `async` - a useful idiom when a hook
needs to await.

### Stamping - as shipped

`PDFAddMark.zip#entry/src/main/ets/pages/AddMarkPage.ets`

```ts
Button($r('app.string.mark'))
  .onClick(async () => {
    if (this.isMarked) {
      this.promptAction.showToast({ message: $r('app.string.isMarked'), duration: 2000 });
      return;
    }
    this.outPdfPath = this.context.filesDir + '/' + Date.now() + Constants.PDF_NAME;
    let res = this.pdfDocument.loadDocument(this.filePath);
    if (res === pdfService.ParseResult.PARSE_SUCCESS) {
      let pdfPage: pdfService.PdfPage = this.pdfDocument.getPage(0);
      let imgPath = this.context.filesDir + '/' + Constants.MARKER_NAME;
      pdfPage.addImageObject(imgPath, 370, 200, 120, 120);
      let result = this.pdfDocument.saveDocument(this.outPdfPath);
      Logger.info(TAG, 'PdfPage', 'addImageObject %{public}s!', result ? 'success' : 'fail');
      this.isMarked = true;
    }
    this.pdfDocument.releaseDocument();      // runs even on a failed load - HW-05-0071
    this.reloadDocument(this.outPdfPath);    // reloads a file that may not exist
  })

async reloadDocument(outPdfPath: string) {
  this.controller.releaseDocument();
  let loadResult: pdfService.ParseResult = await this.controller.loadDocument(outPdfPath);
  if (pdfService.ParseResult.PARSE_SUCCESS === loadResult) {
    Logger.info(TAG, 'aboutToAppear', 'PdfView Component has been successfully loaded!');
  }
}
```

Corrected control flow:

```ts
.onClick(async () => {
  if (this.isMarked) {
    this.promptAction.showToast({ message: $r('app.string.isMarked'), duration: 2000 });
    return;
  }
  const candidatePath = this.context.filesDir + '/' + Date.now() + Constants.PDF_NAME;
  const res = this.pdfDocument.loadDocument(this.filePath);
  if (res !== pdfService.ParseResult.PARSE_SUCCESS) {
    this.pdfDocument.releaseDocument();
    this.promptAction.showToast({ message: $r('app.string.mark_failed'), duration: 2000 });
    return;
  }
  const pdfPage: pdfService.PdfPage = this.pdfDocument.getPage(0);
  pdfPage.addImageObject(this.context.filesDir + '/' + Constants.MARKER_NAME, 370, 200, 120, 120);
  const saved = this.pdfDocument.saveDocument(candidatePath);
  this.pdfDocument.releaseDocument();
  if (!saved) {
    this.promptAction.showToast({ message: $r('app.string.mark_failed'), duration: 2000 });
    return;
  }
  this.outPdfPath = candidatePath;
  this.isMarked = true;
  await this.reloadDocument(this.outPdfPath);
})
```

### Saving through the document picker - as shipped

`PDFAddMark.zip#entry/src/main/ets/pages/AddMarkPage.ets`

```ts
async callFilePickerSaveFile(): Promise<void> {
  try {
    let documentSaveOptions = new picker.DocumentSaveOptions();
    documentSaveOptions.newFileNames = [Constants.PDF_NAME];
    let documentPicker = new picker.DocumentViewPicker();
    documentPicker.save(documentSaveOptions).then((documentSaveResult) => {
      if (documentSaveResult !== null && documentSaveResult !== undefined) {
        this.uriSave = documentSaveResult[0];
      }
      FileUtils.copyFile2Document(this.outPdfPath, this.uriSave);   // outside the guard - HW-05-0072
      this.promptAction.showToast({
        message: $r('app.string.save_success'),
        duration: 2000
      });
    }).catch((err: BusinessError) => {
      Logger.error(TAG, 'DocumentViewPicker.save failed with err: ' + JSON.stringify(err));
    });
  } catch (err) {
    Logger.error(TAG, 'DocumentViewPicker failed with err: ' + err);
  }
}
```

Corrected result handling:

```ts
documentPicker.save(documentSaveOptions).then((documentSaveResult) => {
  if (!documentSaveResult || documentSaveResult.length === 0) {
    return;                                    // the user cancelled
  }
  this.uriSave = documentSaveResult[0];
  const copied = FileUtils.copyFile2Document(this.outPdfPath, this.uriSave);
  this.promptAction.showToast({
    message: copied ? $r('app.string.save_success') : $r('app.string.save_failed'),
    duration: 2000
  });
}).catch((err: BusinessError) => {
  Logger.error(TAG, 'DocumentViewPicker.save failed with err: ' + JSON.stringify(err));
});
```

### Copying into the picked location - as shipped

`PDFAddMark.zip#entry/src/main/ets/common/FileUtils.ets`

```ts
public static copyFile2Document(src: string, dest: string): boolean {
  if (!fs.accessSync(src)) {
    return false;
  }
  let srcFile = fileIo.openSync(src, fileIo.OpenMode.READ_WRITE);
  let destFile = fileIo.openSync(dest, fileIo.OpenMode.READ_WRITE);
  fileIo.copyFileSync(srcFile.fd, destFile.fd);
  fileIo.closeSync(srcFile);
  fileIo.closeSync(destFile);
  return true;
}
```

Corrected:

```ts
public static copyFile2Document(src: string, dest: string): boolean {
  if (!fs.accessSync(src)) {
    return false;
  }
  let srcFile: fileIo.File | undefined = undefined;
  let destFile: fileIo.File | undefined = undefined;
  try {
    srcFile = fileIo.openSync(src, fileIo.OpenMode.READ_ONLY);
    destFile = fileIo.openSync(dest, fileIo.OpenMode.READ_WRITE);
    fileIo.copyFileSync(srcFile.fd, destFile.fd);
    return true;
  } catch (e) {
    Logger.error(TAG, `copyFile2Document failed: ${JSON.stringify(e)}`);
    return false;
  } finally {
    if (srcFile) {
      fileIo.closeSync(srcFile);
    }
    if (destFile) {
      fileIo.closeSync(destFile);
    }
  }
}
```

## Permissions & config

`PDFAddMark.zip#entry/src/main/module.json5` declares **no `requestPermissions`
block**, and that is correct for this scenario:

- The source PDF and the seal image come from `resources/rawfile`, which the app
  owns.
- The stamped copy is written into `context.filesDir`, the app's own sandbox.
- The final save target comes from `DocumentViewPicker.save`, which grants access
  to exactly the file the user chose.

The document likewise has no 权限说明 section - verified consistent.

`PDFAddMark.zip#build-profile.json5` pins `compatibleSdkVersion` and
`targetSdkVersion` to `6.0.0(20)` and enables
`strictMode: { caseSensitiveCheck: true }`, which is why the project-tree casing
in the document matters (HW-05-0074).

Bundled assets, both referenced through `Constants`:

```ts
export class Constants {
  static readonly TOP_RECT_HEIGHT: string = 'topRectHeight';
  static readonly BOTTOM_RECT_HEIGHT: string = 'bottomRectHeight';
  public static PDF_NAME = '物业小区放假通知书.pdf';
  public static MARKER_NAME = 'mark.png';
}
```

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **The viewer and the editor are separate.** `pdfViewManager.PdfController`
  drives the `PdfView`; `pdfService.PdfDocument` performs the edit. They must be
  loaded and released independently, and the editor's output only reaches the
  screen after the viewer reloads it.
- **`addImageObject` takes absolute sandbox coordinates.** Its signature is
  `addImageObject(path: string, x: number, y: number, width: number, height: number): void`
  and the image path must already exist in the sandbox - which is why the seal is
  copied out of `rawfile` at startup.
- **`saveDocument` returns a boolean, not a promise.** The official guide's own
  examples log `result ? 'success' : 'fail'`; treat `false` as a real failure
  path.
- **`loadDocument` returns a `ParseResult`.** Always compare against
  `pdfService.ParseResult.PARSE_SUCCESS` rather than assuming success.
- **Stamping is once-only in this sample.** `isMarked` gates the button, and the
  seal position (`370, 200`) and size (`120 x 120`) are hard-coded for page 0 of
  the bundled document; nothing here generalises to arbitrary page sizes.
- **The original is preserved.** `outPdfPath` gets a `Date.now()` prefix, so each
  stamp writes a new sandbox file and `filePath` is never overwritten.
- **`terminateSelf` is the back action.** The header's back button ends the
  ability rather than popping a navigation stack - this sample has a single page
  and no `Navigation`.

## Pitfalls

- **Releasing the document and reloading the viewer outside the success branch is
  incorrect.** `outPdfPath` is reassigned before anything is written there, so a
  failed `loadDocument` - or a `saveDocument` that returned `false` - leaves the
  viewer reloading a file that does not exist, while `isMarked` still reports
  success. Move both calls inside the branch and act on the save result.
  (HW-05-0071)
- **Checking only `null`/`undefined` on the picker result is incorrect.** A
  cancelled picker yields an empty array, so `uriSave` keeps its old value; the
  copy then runs outside the guard and the success toast fires regardless of
  `copyFile2Document`'s return value. Return early on an empty array and branch on
  the boolean. (HW-05-0072)
- **`copyFile2Document` without `try`/`finally` is incorrect.** A throw between
  the two `openSync` calls leaks the source descriptor; a throw in
  `copyFileSync` leaks both. The sibling method in the same class already shows
  the guarded pattern, and the source should be opened `READ_ONLY`.
  (HW-05-0073)
- **The project tree's root `ets/src/main/ets` is incorrect** - the module is
  `entry` - and `addMarkPage.ets` is mis-cased relative to the shipped
  `AddMarkPage.ets` while the build enables `caseSensitiveCheck`. (HW-05-0074)
- **The document's `PageFit.FIT_WIDTH` does not match the sample's
  `PageFit.FIT_NONE`,** and its save snippet drops the `Logger.error` calls the
  sample performs, teaching empty handlers for exactly the failures HW-05-0072
  depends on. (HW-05-0075)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-guides/07_application-services/pdf-add-txt-img-annot.md` -
  the `addImageObject(path, x, y, width, height): void` signature, the
  `loadDocument` / `getPage` / `saveDocument` sequence, and the
  `result ? 'success' : 'fail'` convention for `saveDocument`'s boolean.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/pdf-add-txt-img-annot
- `documentation/harmonyos-guides/07_application-services/pdf-pdfview-component.md` -
  `PdfController` construction and `registerPageCountChangedListener`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/pdf-pdfview-component
- `documentation/harmonyos-references/02_application-framework/js-apis-file-picker.md` -
  `DocumentViewPicker.save` and `DocumentSaveOptions.newFileNames`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-picker
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `OpenMode` flags, `accessSync`, `copyFileSync` and `closeSync`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/06_application-services/pdf-arkts-pdfservice.md`
  and `pdf-arkts-pdfviewmanage.md` - the PDF Kit reference pages named by the
  document; both are stubs in this repository, so the API details above were
  taken from the PDF Kit guides instead.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/pdf-arkts-pdfservice
