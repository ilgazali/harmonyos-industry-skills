---
id: EDU-20
title: Scan and submit homework - DocumentScanner to a PDF in the app sandbox
industry: 04_education
doc: huawei_industry_tree/04_education/docs/20_scan_submit_homework.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/scan_submit_homework-0000002337721418
sample: huawei_industry_tree/04_education/downloads/SubmitHomework.zip
kits: ["@kit.VisionKit", "@kit.CoreFileKit", "@kit.PDFKit", "@kit.ArkUI", "@kit.ArkTS", "@kit.PerformanceAnalysisKit"]
apis: [DocumentScanner, DocumentScannerConfig, SaveOption, FilterId, ShootingMode, maxShotCount, isGallerySupported, isShareable, originalUris, onResult, "fileIo.accessSync", "fileIo.openSync", "fileIo.readSync", "fileIo.writeSync", "fileIo.unlinkSync", "fileIo.copyFile", "fileIo.lstatSync", ReadOptions, WriteOptions, Navigation, NavPathStack, pushPath, onPop, "@Observed", "@ObjectLink", "util.generateRandomUUID"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-04-0147, HW-04-0148, HW-04-0149, HW-04-0150, HW-04-0151, HW-04-0152, HW-04-0153, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card when the user must **turn paper into a document inside your app** -
homework, a signed form, a receipt, an ID page. `DocumentScanner` from Vision Kit
does the whole capture experience: camera, edge detection, perspective
correction, filters, multi-page capture and PDF assembly.

The two things worth knowing before you start:

- **It needs no permission at all.** `DocumentScanner` is a system control, like
  `SaveButton` in `EDU-08`: the user's interaction with it *is* the
  authorisation. Do not declare `ohos.permission.CAMERA` for this - compare
  `EDU-16`, which builds a camera preview by hand and needs the permission and a
  full request flow.
- **The result is a URI you have a moment to act on.** `onResult` hands back
  URIs owned by the scanner; copy what you need into your own sandbox
  immediately.

The scanner integration itself is ten lines of configuration. Everything that
goes wrong in this sample goes wrong afterwards, in the copy.

## Feature checklist

- A homework list showing each assignment's course, class, publication date and
  days remaining.
- Opening an assignment offers 扫描上传; tapping it launches the document
  scanner.
- The scanner is limited to 30 pages, manual shooting, no gallery import,
  original (unfiltered) output, saving as a single PDF.
- A completed scan copies the PDF into the app sandbox under the assignment's own
  path and returns to the detail page with the submission marked present.
- The submitted PDF can be previewed and deleted.
- Cancelling the scanner returns without changing anything.

## Architecture

One `entry` module, four pages, one utility file.

```
entry/src/main/ets
├── common/Constants.ets            layout sizes
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── pages
│   ├── MainPage.ets                @Entry - the homework list, builds HomeworkDetail[]
│   ├── SubmitHomeworkDetail.ets    one assignment: scan, preview, delete
│   ├── DocScan.ets                 the DocumentScanner destination
│   └── PdfPreview.ets              PdfView over the submitted file
├── util/HomeworkDetail.ets         HomeworkDetail, PageParam, readWriteFile, getFileSize
└── views/HomeworkListView.ets      one row in the list
```

The documented tree matches the zip.

**`PageParam` is both the navigation argument and the return value,** and that is
the design decision worth copying:

```typescript
@Observed
export class PageParam {
  index: number;
  homeworkDetail: HomeworkDetail;
  pdfFileExist: boolean = false;
  isPdfDelClick: boolean = false;
}
```

`SubmitHomeworkDetail` pushes the scanner with
`pushPath({ name: 'DocScan', param: this.pageParam, onPop: () => {...} })`, the
scanner mutates the same object and pops it back, and the detail page's `onPop`
re-reads it. One object travels down and back, so there is no result-parsing and
no shared singleton - the comment in the source puts it exactly: *"The parameter
returned by onPop is the same as that returned by param."*

**The scanner's whole contract is `DocumentScannerConfig` in and `onResult`
out.** There is no controller, no lifecycle, nothing to release.

**Everything after `onResult` is hand-written file work,** and that is where all
seven findings are - an inverted existence check that stops the copy from ever
running (`HW-04-0147`), a `finally` that closes null handles (`HW-04-0148`), and
an async delete racing a synchronous create (`HW-04-0149`).

## Implementation steps

1. **Declare no permission.** `DocumentScanner` handles the camera itself.
2. **Configure the scanner in `aboutToAppear`** - see the snippet below. The
   options that matter are `saveOptions`, `maxShotCount` and
   `isGallerySupported`.
3. **Give each assignment its own destination path** before the scan, so the
   result has somewhere to go.
4. **Branch on the result code explicitly** - success, cancel, and everything
   else - and check `uris.length` before indexing (`HW-04-0151`).
5. **Copy with `fs.copyFile`**, not a hand-rolled 4 KB loop, and not on the UI
   thread if the document can be large (`HW-04-0152`).
6. **Delete the previous submission with `unlinkSync`,** so the delete completes
   before the create (`HW-04-0149`).
7. **Report copy failures.** Return a boolean and mark the submission only on
   success (`HW-04-0153`).
8. **Pop with the mutated `PageParam`** so the detail page sees the result
   through its `onPop`.

## Verified snippets

All snippets are from `SubmitHomework.zip`. Corrected forms are marked.

**Configuring the scanner — `entry/src/main/ets/pages/DocScan.ets`** (as shipped)

```typescript
import { DocumentScanner, DocumentScannerConfig, FilterId, SaveOption, ShootingMode } from '@kit.VisionKit';

private docScanConfig = new DocumentScannerConfig();

aboutToAppear() {
  this.docScanConfig.isGallerySupported = false;      // homework must be photographed, not uploaded
  this.docScanConfig.maxShotCount = 30;               // up to 30 pages in one document
  this.docScanConfig.defaultFilterId = FilterId.ORIGINAL;   // no whitening - keep the teacher's marks
  this.docScanConfig.defaultShootingMode = ShootingMode.MANUAL;
  this.docScanConfig.isShareable = true;
  this.docScanConfig.originalUris = [];
  this.docScanConfig.saveOptions = [SaveOption.PDF];  // one PDF, not N images
}
```

**`saveOptions: [SaveOption.PDF]` is what makes `uris` a single-element array.**
With `SaveOption.IMAGE` the scanner returns one URI per page and the caller has
to assemble them; asking for PDF means the control does the multi-page assembly
and hands back one file - which is why the rest of the code only ever looks at
`uris[0]`.

**`isGallerySupported = false` is a policy decision, not a technical one.**
Homework must be photographed now; allowing a gallery import would let a student
submit a picture taken at any time. Turning it off in the scanner is the right
place for that rule.

**`FilterId.ORIGINAL` rather than a whitening filter** keeps pencil work and
teacher annotations legible - the enhancement filters are tuned for printed text
and can erase faint handwriting.

**Handling the result — same file** (corrected, see `HW-04-0147`, `HW-04-0151`, `HW-04-0153`)

```typescript
DocumentScanner({
  scannerConfig: this.docScanConfig,
  onResult: (code: number, saveType: SaveOption, uris: string[]) => {
    hilog.info(0x0001, TAG, `result code: ${code}, save: ${saveType}`);
    if (code === -1) {                                  // user cancelled
      this.pageInfos.pop();
      return;
    }
    // FIX: the sample has no failure branch - any non -1 code falls through to the
    //      success path and indexes uris[0] with no length check.
    if (code !== 0 || uris.length === 0 || !this.pageParam) {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.scan_failed') });
      this.pageInfos.pop();
      return;
    }
    // FIX: the sample writes `let res = fileIo.accessSync(uris[0]); if (!res && ...)`,
    //      and accessSync returns TRUE when the file exists - so the copy runs only
    //      when the scan produced nothing.
    if (!readWriteFile(uris[0], this.pageParam.homeworkDetail.pdfFileUri)) {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.save_failed') });
      return;
    }
    this.pageParam.pdfFileExist = true;
    this.pageParam.isPdfDelClick = false;
    this.pageInfos.pop(this.pageParam);                  // return the mutated param
  }
})
```

**`pop(this.pageParam)` is the return path, and the document omits it**
(`HW-04-0150`). The detail page pushed this destination with an `onPop` callback
over the same object, so mutating the object and popping is how the result gets
home - there is no other channel.

**The inverted guard is the defect that makes the feature not work at all.**
`accessSync` is documented as *"The value true means the file exists"*, and the
comment directly above the branch says "The source file is accessible" - so the
negation contradicts both the API and the author's own stated intent.

**Copying into the sandbox — `entry/src/main/ets/util/HomeworkDetail.ets`** (corrected, see `HW-04-0148`, `HW-04-0149`, `HW-04-0152`, `HW-04-0153`)

```typescript
// FIX: the sample hand-rolls a 4 KB read/write loop and returns void.
export function readWriteFile(srcFilePath: string, dstFilePath: string): boolean {
  try {
    if (fs.accessSync(dstFilePath)) {
      fs.unlinkSync(dstFilePath);        // FIX: sample calls the async fs.unlink and does not await
    }
    fs.copyFile(srcFilePath, dstFilePath);   // FIX: replaces the whole chunk loop
    return true;
  } catch (error) {
    hilog.error(0x0001, 'HomeworkDetail',
      `copy failed: ${(error as BusinessError).code}`);   // FIX: sample's catch is empty
    return false;
  }
}
```

For reference, the shipped loop - the shape to recognise and replace:

```typescript
let srcFile: fs.File = null!;                 // both handles start as null
let destFile: fs.File = null!;
try {
  srcFile = fs.openSync(srcFilePath, fs.OpenMode.READ_ONLY);
  destFile = fs.openSync(dstFilePath, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
  let bufSize = 4096;
  let readSize = 0;
  let buf = new ArrayBuffer(bufSize);
  let readOptions: ReadOptions = { offset: readSize, length: bufSize };
  let readLen = fs.readSync(srcFile.fd, buf, readOptions);
  while (readLen > 0) {
    readSize += readLen;
    fs.writeSync(destFile.fd, buf, { length: readLen } as WriteOptions);
    readOptions.offset = readSize;            // advance the READ offset each pass
    readLen = fs.readSync(srcFile.fd, buf, readOptions);
  }
} catch (error) {
  // Customized error handling            <- empty; see HW-04-0153
} finally {
  fs.close(srcFile);                        // <- throws when the open failed; see HW-04-0148
  fs.close(destFile);
}
```

**Three things to take from the loop even though you should not use it.**
`ReadOptions.offset` must be advanced by the bytes read so far - it is an
absolute position, not a delta. `WriteOptions.length` must be `readLen`, not
`bufSize`, or the final partial chunk writes garbage past the end. And the
`finally` must null-check: `null!` initialisers plus `fs.close(null)` turn a
failed open into a second, unrelated error thrown from the error handler.

**`fs.copyFile` does all of it in one call.** The chunk loop exists in a lot of
sample code and is almost never the right answer.

**Passing state down and back — `entry/src/main/ets/pages/SubmitHomeworkDetail.ets`** (as shipped)

```typescript
// The parameter returned by onPop is the same as that returned by param.
this.pageInfos.pushPath({
  name: 'DocScan',
  param: this.pageParam,
  onPop: () => {
    // this.pageParam has been mutated in place by the scanner page
  }
});
```

**One `@Observed` object as both argument and result** avoids the
`getParamByName` misuse seen in `EDU-09` and `EDU-14`: the destination receives
the object directly through its builder parameter, mutates it, and pops. Because
`PageParam` is `@Observed` and the detail page holds it, the UI updates without
the `onPop` body having to do anything.

## Permissions & config

**None.** `entry/src/main/module.json5` declares no `requestPermissions` block,
and that is correct: `DocumentScanner` is a system control that owns the camera
session for the duration of the scan. Declaring `ohos.permission.CAMERA` here
would be the same mistake as declaring `WRITE_IMAGEVIDEO` alongside a
`SaveButton` (`EDU-08`, `HW-04-0049`).

Destination paths are generated per assignment in `MainPage`:

```typescript
new HomeworkDetail('语文', false, '小班1班', publishDate, expireDate, true,
  this.context.filesDir + '/' + util.generateRandomUUID(true).toString() + '.pdf')
```

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `DocumentScanner` is a Vision Kit control with its own device support matrix;
  it is not available everywhere the rest of the SDK is.
- **The destination path is a fresh UUID on every launch.** `MainPage` builds its
  `HomeworkDetail` list at construction with `generateRandomUUID`, so a PDF
  written in one session is orphaned in `filesDir` on the next and the assignment
  shows as unsubmitted again. Nothing is persisted.
- Up to 30 pages per scan (`maxShotCount`), saved as one PDF; the sample only
  ever reads `uris[0]`, so switching to `SaveOption.IMAGE` would need a different
  copy path.
- The homework list is two literal `HomeworkDetail` objects; there is no backend,
  and "submitting" means copying a file into the sandbox.
- `getFileSize` reports in MB rounded to two decimals via `lstatSync`, so a file
  under 10 KB displays as `0`.
- The copy runs on the UI thread inside the scanner callback (`HW-04-0152`).

## Pitfalls

- **`HW-04-0147` — the existence check is inverted,** so the scanned PDF is
  copied only when it does not exist. `accessSync` returns `true` when the file
  is there. This is the defect that makes the feature not work.
- **`HW-04-0148` — the copy's `finally` closes two `null!` handles** when the
  open fails, throwing a second error out of the error path.
- **`HW-04-0149` — the previous submission is deleted with the async
  `fs.unlink`** and the destination is reopened immediately, so the delete can
  land after the write and remove the new file. Use `unlinkSync`.
- **`HW-04-0150` — the documented `onResult` differs from the shipped one** in
  its success condition, its cancel handling, and the `pop(this.pageParam)` that
  returns the result.
- **`HW-04-0151` — only `code === -1` is handled;** every other non-success code
  takes the success path and indexes `uris[0]` unchecked.
- **`HW-04-0152` — a multi-megabyte PDF is copied synchronously in 4 KB chunks**
  on the UI thread. `fs.copyFile` replaces the whole loop.
- **`HW-04-0153` — the copy's `catch` is empty,** so a failed write still marks
  the homework submitted.
- **Do not declare a camera permission for `DocumentScanner`.** The control is
  the grant.

## References

- `documentation/harmonyos-references/07_ai/vision-document-scanner.md` - `DocumentScanner`, `DocumentScannerConfig`, `SaveOption`, `FilterId`, `ShootingMode`, and the `onResult` codes
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/vision-document-scanner
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `accessSync` returning true when the file exists, `copyFile`, `unlinkSync`, `ReadOptions`/`WriteOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `pushPath` with `param` and `onPop`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-observed-and-objectlink.md` - `@Observed` on a class shared between pages
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-observed-and-objectlink
- `EDU-08` - the same "system control instead of a permission" principle, with `SaveButton`
- `EDU-16` - what building the camera by hand costs, for contrast
- `EDU-09` - the same null-handle-in-`finally` defect, and the `getParamByName` misuse this sample avoids
