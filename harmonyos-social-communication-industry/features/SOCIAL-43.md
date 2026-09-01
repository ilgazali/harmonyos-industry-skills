---
id: SOCIAL-43
title: Send an encrypted file in chat - a DLP-aware DocumentViewPicker round trip with a save-back dialog
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/43_chat_page_file_encryption.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/chat_page_file_encryption-0000002424613385
sample: huawei_industry_tree/14_social_communication/downloads/ChatPageFileEncryption.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit", "@kit.PreviewKit", "@kit.ScenarioFusionKit"]
apis: [atomicService, common, fileIo, filePreview, fileUri, hilog, picker, window, "picker.DocumentSelectOptions", isEncryptionSupported, "DocumentViewPicker.select", "DocumentViewPicker.save", "picker.DocumentSaveOptions", "filePreview.openPreview", "@CustomDialog", createAnimator]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0082, HW-14-0018, HW-14-0085, HW-14-0087]
status: verified-with-fixes
---

## When to use

**Load this card when a chat has to carry documents and some of them are
rights-protected.** HarmonyOS exposes file encryption (DLP) through one flag on
the picker - `DocumentSelectOptions.isEncryptionSupported` - which turns the
system file manager into a picker that can encrypt on the way out. The app
never encrypts anything itself; it opts in, receives a `.dlp` URI, and treats
that suffix as a first-class file type from then on.

The rest of the sample is the plumbing every document-sending feature needs and
most demos skip: copy the picked file into the app sandbox before referencing
it, keep a `FileInfo` per message with name, size, suffix and icon, and branch
on click - preview an ordinary file in place, but push an encrypted one back out
through `DocumentViewPicker.save` because only the file manager can open it.

It generalises past chat: email attachments, ticket uploads, document sharing in
a workspace app. **Read `HW-14-0082` first** - the shipped select path throws on
the ordinary cancel gesture, and the copy destination is opened without `TRUNC`.

## Feature checklist

- The chat page shows a seeded incoming and outgoing text bubble.
- The `+` button animates the composer open to a menu strip (52 → 147 vp) and
  closed again.
- The 文件 (file) entry launches the system file manager through
  `DocumentViewPicker.select`.
- On API 19 or later the picker is launched with `isEncryptionSupported = true`,
  so the file manager offers 加密分享 (encrypted share); below that it is
  launched plain.
- The picked file is copied into the app sandbox and appended to the list as a
  bubble with icon, decoded name and size in MB.
- Tapping a plain file opens it in the system preview
  (`filePreview.openPreview`).
- Tapping a `.dlp` file opens a custom dialog asking whether to save it
  locally; confirming raises `DocumentViewPicker.save` with the DLP suffix
  list.
- Cancelling the picker returns to the chat with nothing added and nothing
  broken.
- The list scrolls to the bottom whenever a file is added.

## Architecture

One `entry` module, one page, and the file work factored into a `fileUtils`
folder split by API vintage.

```
entry/src/main/ets
├── common/Constants.ets             all sizes, weights, durations as named constants
├── components/CustomDialog.ets      SaveEncryptedFileDialog: the save-to-local confirm
├── entryability/EntryAbility.ets    avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── fileUtils/
│   ├── FileInfo.ets                 name, uri, size, suffix, icon resource name
│   ├── FileUtils.ets                select, copyIntoFile, preview, mime map, icon map
│   └── FileUtilsNewFeatures.ets     isDlpFile + saveDlpFile (the API-20 half)
└── pages/ChatPage.ets               @Entry: bubbles, the animated menu, the click branch
```

The documented tree matches the zip.

**The design decision worth copying** is the split between `FileUtils.ets` and
`FileUtilsNewFeatures.ets`. Everything that works on any supported API lives in
the first; everything that depends on the encryption feature - the `.dlp` suffix
test and the save-back through the picker - lives in the second. The page keeps
one runtime gate,
`atomicService.getSystemInfoSync(['sdkApiVersion']).sdkApiVersion`, and calls
across the boundary only behind it. That is a cleaner shape than sprinkling
version checks through a single utility file, and it makes the "what do I lose
on an older device" question answerable by reading one file.

**The decision worth avoiding** is `FileInfo.imgResource` being a *string*
(`'app.media.pdf'`) that the page then re-resolves with `$r(...)` at render
time. `getImgResourceFromSuffix` returns `''` for any unmapped suffix, and
`$r('')` is not a resource - so the icon lookup is a stringly-typed contract
with a failure mode that only shows up at paint time. Store a `Resource`.

## Implementation steps

1. **Read the device API level once,**
   `atomicService.getSystemInfoSync(['sdkApiVersion']).sdkApiVersion`, and keep
   it in a field.
2. **Set `isEncryptionSupported = true` only above the gate,** and build a plain
   `DocumentSelectOptions` below it - the property does not exist on older
   pickers.
3. **Set `maxSelectNumber = 1`.** The copy path indexes `documentSelectResult[0]`
   and nothing downstream is written for multiple files.
4. **Guard the picker result and attach a `.catch`** (`HW-14-0082`): a cancelled
   select resolves with an empty array.
5. **Copy the picked file into `context.filesDir` before using it.** The picker
   URI is a transient grant; the sandbox copy is what survives into the message
   list.
6. **Open the source read-only and the destination with `TRUNC`**
   (`HW-14-0082`), or re-sending a shorter file leaves the previous copy's tail
   behind.
7. **`decodeURIComponent` the file name** - the picker returns it
   percent-encoded, and Chinese names are unreadable without it.
8. **Treat `.dlp` as a compound suffix.** `report.pdf.dlp` must map to the PDF
   icon, which is why `createFileInfo` re-joins the last two segments.
9. **Give the `ForEach` a unique key** (`HW-14-0018`) - the shipped key
   generator is typed `(item: string)` over `FileInfo` objects.
10. **Branch on click:** `isDlpFile` → confirm dialog → `DocumentViewPicker.save`;
    otherwise `filePreview.openPreview`.

## Verified snippets

All snippets are from `ChatPageFileEncryption.zip`. Corrected forms are marked.

**The API gate around the encryption flag — `entry/src/main/ets/pages/ChatPage.ets`** (as shipped)

```typescript
import { atomicService } from '@kit.ScenarioFusionKit';
import { picker } from '@kit.CoreFileKit';

private currentAPIVersion = atomicService.getSystemInfoSync(['sdkApiVersion']).sdkApiVersion;
private context = this.getUIContext().getHostContext() as common.Context;

@Builder
menuBuilder(ic: Resource, text: Resource) {
  Column({ space: Constants.SPACE_8 }) { /* icon tile + label */ }
  .onClick(() => {
    if (this.context.resourceManager.getStringSync(text.id) ===
    this.resourceManager?.getStringSync($r('app.string.file').id)) {
      if (this.currentAPIVersion && this.currentAPIVersion >= 19) {
        let documentSelectOptions = new picker.DocumentSelectOptions();
        documentSelectOptions.maxSelectNumber = 1;
        documentSelectOptions.isEncryptionSupported = true;     // the whole feature
        selectFile(this.context, documentSelectOptions, this.files);
      } else {
        let documentSelectOptions = new picker.DocumentSelectOptions();
        documentSelectOptions.maxSelectNumber = 1;
        selectFile(this.context, documentSelectOptions, this.files);
      }
    }
  });
}
```

**One property is the entire encryption integration.** With
`isEncryptionSupported` set, the system file manager shows its 加密分享
(encrypted share) button; the user authenticates with a Huawei account, the file
manager produces the protected copy, and what comes back to the app is an
ordinary URI ending `.dlp`. The app holds no keys, calls no crypto API and
requires no permission for any of it - which is exactly why this is the
recommended route rather than encrypting in-app.

The gate is `>= 19` in code while the document's 约束与限制 says the feature
shows only from API 20; the property is simply absent below the gate, so the
else-branch exists to keep the file button working rather than to offer a
lesser encryption.

Note the branch condition: the builder is shared by a menu strip that today has
one entry, and it identifies itself by resolving `text.id` back to a string and
comparing it with `$r('app.string.file')`. That works, but it is a string
comparison standing in for an enum - give menu entries an id if the strip grows.

**Select and copy — `entry/src/main/ets/fileUtils/FileUtils.ets`** (corrected, see `HW-14-0082`)

```typescript
export function selectFile(context: common.Context,
  documentSelectOptions: picker.DocumentSelectOptions, files: Array<FileInfo>) {
  let documentPicker = new picker.DocumentViewPicker(context);
  documentPicker.select(documentSelectOptions).then((documentSelectResult: Array<string>) => {
    if (!documentSelectResult || documentSelectResult.length === 0) {
      return;                      // FIX: cancel resolves with an empty array
    }
    let file = createFileInfo(documentSelectResult, context.filesDir);
    files.push(file);
  }).catch((err: BusinessError) => { // FIX: the sample attaches no rejection handler
    hilog.error(0x0000, 'testTag', `select failed. code: ${err.code}, message: ${err.message}`);
  });
}

// copy file and return file sizes
export function copyIntoFile(srcDir: string, destFileUri: string): number {
  let srcFile = fs.openSync(srcDir, fs.OpenMode.READ_ONLY);              // FIX: was READ_WRITE|CREATE
  let destFile = fs.openSync(destFileUri,
    fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE | fs.OpenMode.TRUNC);    // FIX: TRUNC added
  let readSize = 0;
  try {
    let bufSize = 4096;
    let buf = new ArrayBuffer(bufSize);
    let readOptions: ReadOptions = { offset: readSize, length: bufSize };
    let readLen = fs.readSync(srcFile.fd, buf, readOptions);
    while (readLen > 0) {
      readSize += readLen;
      let writeOptions: WriteOptions = { length: readLen };
      fs.writeSync(destFile.fd, buf, writeOptions);
      readOptions.offset = readSize;
      readLen = fs.readSync(srcFile.fd, buf, readOptions);
    }
  } catch (err) {
  } finally {
    fs.closeSync(srcFile);
    fs.closeSync(destFile);
  }
  return readSize;
}
```

**Both fixes are about the ordinary path, not the exotic one.** Cancelling a
picker is a routine gesture, and `select()` honours it by resolving with an
empty array - so `documentSelectResult[0].split('/')` inside `createFileInfo`
dereferences `undefined` and throws a `TypeError` with no `.catch` to receive
it. The guard belongs at the call site, before `createFileInfo` runs.

`TRUNC` matters because the destination path is derived from the file *name*:
`filesDir + '/' + destName`. Send `notes.txt`, replace it with a shorter
`notes.txt`, send it again - `CREATE` without `TRUNC` opens the existing file at
offset 0 and overwrites only as many bytes as the new file has, leaving the old
tail in place. The chat then shows a file that is silently a mixture of two.
Opening the *source* `READ_WRITE | CREATE` is the same copy-paste: a picked file
is a read-only grant, and asking for write access can fail outright on a DLP
URI.

The identical pair appears in `FileUtilsNewFeatures.ets:31-34` on the save-back
path, and needs the same treatment.

**Naming, sizing and the compound `.dlp` suffix — same file** (as shipped)

```typescript
export function createFileInfo(documentSelectResult: string[], filesDir: string): FileInfo {
  let fileStrArr = documentSelectResult[0].split('/');
  let fileName = fileStrArr[fileStrArr.length - 1];

  let destName = decodeURIComponent(fileName);
  let srcDir = documentSelectResult[0];
  let destUri = filesDir + '/' + destName;

  let fileSize = Math.round(copyIntoFile(srcDir, destUri) / 10000) / 100.0;
  let suffixStr: string[] = fileName.split('.');
  let suffix: string = suffixStr[suffixStr.length - 1];
  if (suffix === 'dlp') {
    suffix = suffixStr[suffixStr.length - 2] + '.' + suffixStr[suffixStr.length - 1];
  }
  let imgResource = getImgResourceFromSuffix(suffix);
  let file = new FileInfo(destName, destUri, fileSize, suffix, imgResource);
  return file;
}
```

**Encryption turns the suffix into two tokens.** `report.pdf` becomes
`report.pdf.dlp`, so a naive "last segment" suffix read yields `dlp` for every
protected file and every icon collapses to one. Re-joining the last two
segments is what lets `getImgResourceFromSuffix` map `pdf.dlp` to the PDF icon,
and it is why that map lists both forms of each type. `isDlpFile` still tests
only the final token, which is the right granularity for the click branch.

The size arithmetic is worth reading twice: `copyIntoFile` returns bytes,
`/ 10000` then `Math.round` then `/ 100` yields two decimals of *hundred*-KB
units, and the bubble labels it `MB`. A 1,048,576-byte file displays as
`104.86MB`. Use `/ (1024 * 1024)` if the number is meant to be read.

**The click branch and the save-back dialog — `entry/src/main/ets/pages/ChatPage.ets`** (as shipped)

```typescript
private encryptionConfirmedDialog: CustomDialogController = new CustomDialogController({
  builder: SaveEncryptedFileDialog({ clickedFile: this.clickedFile })
});

.onClick(() => {
  this.clickedFile = file;
  if (isDlpFile(file.fileName)) {
    this.encryptionConfirmedDialog.open();
  } else {
    preview(decodeURIComponent(file.fileName), fileUri.getUriFromPath(file.uri), this.context);
  }
})
```

```typescript
// entry/src/main/ets/fileUtils/FileUtilsNewFeatures.ets
export function saveDlpFile(clickedFileUri: string, context: common.Context) {
  let documentSaveOptions = new picker.DocumentSaveOptions();
  documentSaveOptions.newFileNames = [decodeURIComponent(getFileNameFromUri(clickedFileUri))];
  documentSaveOptions.fileSuffixChoices =
    ['.pdf.dlp', '.pdf', '.txt', '.txt.dlp', '.doc', '.docs', '.doc.dlp', '.docs.dlp',
     '.xls', '.xls.dlp', '.xlsx', '.xlsx.dlp'];
  let documentViewPicker = new picker.DocumentViewPicker(context);
  documentViewPicker.save(documentSaveOptions).then((documentSaveResult: Array<string>) => {
    /* copy sandbox file -> chosen destination */
  }).catch(() => {
  });
}
```

**An encrypted file cannot be previewed in-app, so the app hands it back to the
system.** That is the reason for the whole dialog: `filePreview.openPreview`
has no rights to a DLP payload, while the file manager does. `newFileNames`
pre-fills the save sheet with the original name and `fileSuffixChoices`
enumerates the twelve accepted document forms - each plain type and its `.dlp`
twin - which is also the sample's de-facto statement of what it supports.
Media files are excluded by design; the document says so explicitly.

`@Link clickedFile` in the dialog is what makes one long-lived
`CustomDialogController` usable for every file in the list: the page assigns
`this.clickedFile` before `open()`, and the link carries it in.

## Permissions & config

**None.** The sample declares no `requestPermissions` - correct, and the point
of routing through the picker. `DocumentViewPicker` runs the system file
manager in its own process and returns a URI already granted to this app, so no
`READ_WRITE_DOCUMENT` or storage permission is needed, and the encryption is
performed by the file manager under the user's own account.

`deviceTypes` is `phone`, `tablet`, `2in1` - wider than most samples in this
pack, which fits a document-centric feature.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Encryption applies to document types only - `.txt`, `.doc`, `.docs`,
  `.xls`, `.xlsx`, `.pdf`. Images, video and audio are out of scope for this
  sample and for the DLP flow it uses.
- The encrypted-share step needs a signed-in Huawei account; it cannot be
  exercised on an unprovisioned device or an emulator without one.
- `maxSelectNumber = 1`: one file per send. `createFileInfo` reads
  `documentSelectResult[0]` and ignores the rest.
- The composer is decorative apart from the file entry - the 发送 button has no
  handler, the two text bubbles are static resources, and the menu strip has one
  icon.
- `copyIntoFile` is fully synchronous on the UI thread with a 4 KB buffer. Fine
  for a demo document, visibly janky for a large one; move it to a task pool if
  files can be big.
- File size is displayed in units that are not MB (see above), and the icon for
  any unmapped suffix resolves to `$r('')`.

## Pitfalls

- **`HW-14-0082`** (B/medium, confirmed): `select().then` dereferences
  `documentSelectResult[0]` with no empty-result guard and no `.catch`, so the
  routine picker-cancel path throws an unhandled `TypeError`; separately the
  copy destination is opened `CREATE` without `TRUNC` (in both `FileUtils.ets`
  and `FileUtilsNewFeatures.ets`), so re-sending a shorter file of the same name
  keeps the previous copy's trailing bytes, and the read-only picked source is
  opened `READ_WRITE | CREATE`. Fix: guard the array length, add a `.catch`, add
  `OpenMode.TRUNC` to the destination and open the source `READ_ONLY`.
- **`HW-14-0018`** (B/medium, confirmed): the file list's key generator is
  `(item: string) => item` over an array of `FileInfo` objects, so every row
  stringifies to `[object Object]` and shares one key. ArkUI's diffing cannot
  tell two file bubbles apart - the second file sent may not render, or may
  reuse the first one's node. This is one of six instances of the same broken-key
  pattern across the chat samples, raised on `SOCIAL-08`. Fix: key on the URI
  plus the index, and type the generator parameter as `FileInfo`.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-file-picker.md` - `DocumentViewPicker.select` / `.save`, `DocumentSelectOptions.isEncryptionSupported`, `DocumentSaveOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-picker
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `openSync`, `OpenMode.TRUNC`, `readSync` / `writeSync`, `ReadOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fileuri.md` - `fileUri.getUriFromPath`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fileuri
- `documentation/harmonyos-references/02_application-framework/ts-methods-custom-dialog-box.md` - `@CustomDialog`, `CustomDialogController`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-methods-custom-dialog-box
- `SOCIAL-08` - the origin of `HW-14-0018`, the systematic `ForEach` key defect
- `SOCIAL-11` - the other encryption demo in this pack, for comparison
