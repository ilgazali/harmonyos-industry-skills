---
id: OFFICE-28
title: Adding and deleting annotations in PDF preview - PdfView, annotation listeners, per-page annotation bookkeeping in an RDB store
industry: 05_office
doc: huawei_industry_tree/05_office/docs/28_add_delete_annotations_in_pdf_preview.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_delete_annotations_in_pdf_preview-0000002384668918
sample: huawei_industry_tree/05_office/downloads/DelAllPdfAnnotation.zip
kits: ["@kit.PDFKit", "@kit.ArkData", "@kit.CoreFileKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: [PdfView, "pdfViewManager.PdfController", "PdfController.loadDocument", "PdfController.saveDocument", "PdfController.enableAnnotation", "PdfController.disableAnnotation", "PdfController.deleteSelectedAnnotation", "PdfController.registerPageCountChangedListener", "PdfController.registerAnnotationChangedListener", "PdfController.registerAnnotationSelectedListener", "pdfService.PdfDocument", "PdfDocument.loadDocument", "PdfDocument.getPageCount", "PdfDocument.getPage", "PdfPage.getAnnotations", "pdfService.ParseResult", "pdfService.PageFit", "pdfViewManager.SupportedAnnotationType", "relationalStore.getRdbStore", "RdbStore.executeSql", "RdbStore.query", "RdbStore.insert", "RdbStore.update", "relationalStore.RdbPredicates", "ResultSet.close", "resourceManager.getRawFileContentSync", "fileIo.accessSync", "fileIo.copyFileSync", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "AppStorage.link", "@StorageProp"]
permissions: []
min_api: 24
modules: [entry]
findings: [HW-05-0148, HW-05-0149, HW-05-0150, HW-05-0151, HW-05-0152, HW-05-0153, HW-05-0154, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when a document app must let the user **mark up a PDF while
previewing it** - highlight a passage, then later remove everything they added
without touching annotations that were already in the file.

The hard part is not adding the highlight; `enableAnnotation` does that in one
call. The hard part is **knowing which annotations are the user's own**, because
PDF Kit deletes by index and gives no way to ask "was this annotation here when
the document was opened?". The sample answers that with a small relational
store: for each page it records how many annotations the original document had
(`rawAnnNumber`) and how many the user has added since (`manualAnnNumber`).
Deleting "all annotations" then means deleting `manualAnnNumber` annotations
starting at index `rawAnnNumber`.

Related cards: OFFICE-16 loads and previews a PDF with the same `PdfView`
component; this one adds the editing and bookkeeping layer on top.

## Feature checklist

The application must be able to:

- Copy a bundled PDF into the sandbox on first launch.
- Count the annotations already present on each page and persist those counts.
- Preview the document with `PdfView`.
- Enter highlight-annotation mode on demand and leave it automatically once an
  annotation has been added.
- Keep a per-page count of the annotations the user added.
- Save the document, overwriting the source file, and persist the counts.
- Delete every user-added annotation on every page, then save so the deletion
  survives a restart.
- Tell the user when there is nothing to delete.

## Architecture

Single `entry` HAP, four ArkTS files, no permissions:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | full-screen layout, avoid-area heights into `AppStorage` |
| `entrybackupability/EntryBackupAbility.ets` | backup extension stub |
| `pages/Index.ets` | `@Entry`; `PdfView`, the save button, the delete-all button, the enter-annotation-mode button |
| `toolability/PdfKitAbility.ets` | everything else - sandbox seeding, RDB creation, initial counting, listener registration, delete-all, save-all |

The `ANNOTATION` table is the whole design:

```sql
CREATE TABLE IF NOT EXISTS ANNOTATION (
  ID INTEGER PRIMARY KEY AUTOINCREMENT,
  pageIndex INTEGER,
  rawAnnNumber INTEGER,      -- annotations already in the file when first opened
  manualAnnNumber INTEGER    -- annotations the user has added and saved
)
```

Two independent readers of the same PDF exist at once:

```
resources/rawfile/a.pdf
   |  getRawFileContentSync + fileIo.writeSync   (first launch only)
   v
filesDir/a.pdf  ------------------------------------.
   |                                                 |
   | pdfService.PdfDocument.loadDocument             | fileIo.copyFileSync
   | (counts raw annotations once, in aboutToAppear) v
   |                                            tempDir/temp<rand>.pdf
   |                                                 |
   |                                                 | PdfController.loadDocument
   |                                                 v
   |                                            PdfView on screen
   |                                                 |
   '<------------- controller.saveDocument(filesDir/a.pdf) -- overwrites the source
```

The controller edits a **temporary copy** and saves back over the original. That
is the official "save by overwriting the source document" pattern
(`pdf-open-document.md`), and it is why the counting pass may safely open the
original file directly.

Annotation flow:

```
Button 添加高亮批注  -> controller.enableAnnotation(HIGHLIGHT, 0xFFEEEAAA)
                        isAnnotationMode = true
user drags over text -> registerAnnotationChangedListener(controlType 0 = ADD)
                        manualAnnNumRecord[page]++
                        controller.disableAnnotation()   // auto-exit
Button 保存          -> disableAnnotation if still in annotation mode
                        controller.saveDocument(filePath)
                        saveAllAnnotation: DB manualAnnNumber += manualAnnNumRecord
Button 删除所有批注   -> delAllAnn: for each row with manualAnnNumber != 0,
                          deleteSelectedAnnotation(rawAnnNumber, pageIndex) x N
                          DB manualAnnNumber = 0
                        controller.saveDocument(filePath)
```

`controlType` is `0 = ADD`, `1 = MOD`, `2 = DEL`; the sample names them in
`annotationControlType: string[] = ['ADD', 'MOD', 'DEL']` and only reacts to ADD.

## Implementation steps

1. **Seed the sandbox.** On first launch read `resources/rawfile/a.pdf` with
   `resourceManager.getRawFileContentSync` and write it to `filesDir` with
   `fileIo`. The document credits `@ohos.file.fs` with this read; it is
   ResourceManager that supplies the bytes (HW-05-0153).
2. **Create the store and the table** with `relationalStore.getRdbStore` +
   `executeSql`.
3. **Count the original annotations once.** Load the sandbox file with a
   `pdfService.PdfDocument`, walk `getPageCount()` pages, and insert one row per
   page with `rawAnnNumber = page.getAnnotations().length` and
   `manualAnnNumber = 0`. Skip the whole pass when the table already has rows.
   Build a **fresh `RdbPredicates` per query** (HW-05-0150), **close every
   `ResultSet`** (HW-05-0149), and **release the document** when the count is
   done (HW-05-0151).
4. **Register the listeners before loading into the controller.**
   `registerPageCountChangedListener` must be called once before
   `loadDocument`, per the official note in the `PdfView` reference.
5. **Copy to a temp file and load it into the controller.** Do not set the
   preview mode immediately after loading - the official example warns against
   it in the same place.
6. **Track additions in memory**, and immediately leave annotation mode so the
   next drag scrolls the document instead of highlighting it.
7. **Persist on save**, and **also persist on add** if delete-all must work
   before the first save (HW-05-0148).
8. **Delete by index range**: `deleteSelectedAnnotation(rawAnnNumber, pageIndex)`
   repeated `manualAnnNumber` times removes the user's annotations one after
   another, because each deletion shifts the next one down into
   `rawAnnNumber`.
9. **Save after deleting** - otherwise the deleted annotations come back on the
   next launch, since the file on disk still contains them.

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Seeding the sandbox from rawfile

`DelAllPdfAnnotation.zip#entry/src/main/ets/toolability/PdfKitAbility.ets`

```ts
export function getSandAndWritePdf(context: Context) {
  let filePath: string = context.filesDir + '/a.pdf';
  let res = fileIo.accessSync(filePath);
  if (!res) {
    // 确保在工程目录src/main/resources/rawfile里存在a.pdf文档
    let content: Uint8Array = context.resourceManager.getRawFileContentSync('a.pdf');
    let fdSand = fileIo.openSync(filePath, fileIo.OpenMode.WRITE_ONLY | fileIo.OpenMode.CREATE | fileIo.OpenMode.TRUNC);
    fileIo.writeSync(fdSand.fd, content.buffer);
    fileIo.closeSync(fdSand.fd);
  }
}
```

The path passed to `getRawFileContentSync` is **relative to `resources/rawfile`**
- `'a.pdf'`, not `'rawfile/a.pdf'`. The sample is right and the PDF Kit guide is
wrong about this; see Constraints.

### Creating the bookkeeping table

`DelAllPdfAnnotation.zip#entry/src/main/ets/toolability/PdfKitAbility.ets`

```ts
export async function createPdfAnnRdb(context: Context) {
  const STORE_CONFIG: relationalStore.StoreConfig = {
    name: 'RdbPdfAnn.db',
    securityLevel: relationalStore.SecurityLevel.S3,
    encrypt: false,
    customDir: 'customDir/subCustomDir',
    isReadOnly: false
  };

  const SQL_CREATE_TABLE =
    'CREATE TABLE IF NOT EXISTS ANNOTATION (ID INTEGER PRIMARY KEY AUTOINCREMENT, pageIndex INTEGER, rawAnnNumber INTEGER, manualAnnNumber INTEGER)';

  let store: relationalStore.RdbStore = await relationalStore.getRdbStore(context, STORE_CONFIG);
  await store.executeSql(SQL_CREATE_TABLE);
  return store;
}
```

### Counting the original annotations - as shipped

`DelAllPdfAnnotation.zip#entry/src/main/ets/toolability/PdfKitAbility.ets`

```ts
export async function getAnnNumOfEachPage(filePath: string, pdfDocument: pdfService.PdfDocument,
  store: relationalStore.RdbStore, manualAnnNumRecord: number[]) {
  // 如果数据库已存在数据，则非首次登录，可以不用初始化了
  let predicates = new relationalStore.RdbPredicates('ANNOTATION');
  let resultSet = await store.query(predicates, ['ID', 'pageIndex', 'rawAnnNumber', 'manualAnnNumber']);
  if (resultSet.goToNextRow()) {
    return;                                   // cursor leaked - HW-05-0149
  }

  let loadResult = pdfDocument.loadDocument(filePath);
  if (pdfService.ParseResult.PARSE_SUCCESS === loadResult) {
    let pageCount: number = pdfDocument.getPageCount();
    for (let pageIndex = 0; pageIndex < pageCount; pageIndex++) {
      manualAnnNumRecord[pageIndex] = 0;

      let page: pdfService.PdfPage = pdfDocument.getPage(pageIndex);
      let annotations: Array<pdfService.PdfAnnotation> = page.getAnnotations();
      hilog.info(0x0000, 'testTag', 'current page is %d, raw_annotations_len is %d.', pageIndex, annotations.length);
      // 查询当前页面是否已存入数据库
      predicates.equalTo('pageIndex', pageIndex);      // conditions accumulate - HW-05-0150
      const valueBucket: relationalStore.ValuesBucket = {
        'pageIndex': pageIndex,
        'rawAnnNumber': annotations.length,
        'manualAnnNumber': 0
      };
      try {
        let resultSet = await store.query(predicates, ['ID', 'pageIndex', 'rawAnnNumber', 'manualAnnNumber']);
        if (resultSet.goToNextRow()) {
          // 这里正常走不进来
          let updateNum = await store.update(valueBucket, predicates);
        } else {
          let insertNum = await store.insert('ANNOTATION', valueBucket);
        }
      } catch (e) {
        hilog.error(0x0000, 'testTag', `Failed to update data. Code:${e.code}, message:${e.message}`);
      }
    }
  } else {
    hilog.error(0x0000, 'testTag', 'pdfService loadDocument failed.');
  }
}
```

Corrected shape - fresh predicates, closed cursors, released document:

```ts
export async function getAnnNumOfEachPage(filePath: string, pdfDocument: pdfService.PdfDocument,
  store: relationalStore.RdbStore, manualAnnNumRecord: number[]) {
  const initialised = new relationalStore.RdbPredicates('ANNOTATION');
  const probe = await store.query(initialised, ['ID']);
  try {
    if (probe.goToNextRow()) {
      return;
    }
  } finally {
    probe.close();
  }

  if (pdfDocument.loadDocument(filePath) !== pdfService.ParseResult.PARSE_SUCCESS) {
    hilog.error(0x0000, 'testTag', 'pdfService loadDocument failed.');
    return;
  }
  try {
    const pageCount: number = pdfDocument.getPageCount();
    for (let pageIndex = 0; pageIndex < pageCount; pageIndex++) {
      manualAnnNumRecord[pageIndex] = 0;
      const annotations = pdfDocument.getPage(pageIndex).getAnnotations();
      hilog.info(0x0000, 'testTag', 'page %{public}d has %{public}d raw annotations',
        pageIndex, annotations.length);

      const pagePredicates = new relationalStore.RdbPredicates('ANNOTATION');
      pagePredicates.equalTo('pageIndex', pageIndex);
      const valueBucket: relationalStore.ValuesBucket = {
        'pageIndex': pageIndex,
        'rawAnnNumber': annotations.length,
        'manualAnnNumber': 0
      };
      const existing = await store.query(pagePredicates, ['ID']);
      try {
        if (existing.goToNextRow()) {
          await store.update(valueBucket, pagePredicates);
        } else {
          await store.insert('ANNOTATION', valueBucket);
        }
      } finally {
        existing.close();
      }
    }
  } finally {
    pdfDocument.releaseDocument();
  }
}
```

### Registering the listeners and loading the temp copy

`DelAllPdfAnnotation.zip#entry/src/main/ets/toolability/PdfKitAbility.ets`

```ts
export async function initRegisterEvent(context: Context, filePath: string, controller: pdfViewManager.PdfController,
  annotationControlType: string[], manualAnnNumRecord: number[]) {
  // 该监听方法只能在文档加载前调用一次，非首次打开app需初始化manualAnnNumRecord
  controller.registerPageCountChangedListener((pageCount: number) => {
    AppStorage.setOrCreate('pageCnt', pageCount);
    for (let i = 0; i < pageCount; i++) {
      manualAnnNumRecord[i] = 0;
    }
  });

  let tempFilePath = context.tempDir + `/temp${Math.random()}.pdf`;
  fileIo.copyFileSync(filePath, tempFilePath);
  // 加载临时文件
  let loadResult: pdfService.ParseResult = await controller.loadDocument(tempFilePath);
  // 注意：这里刚加载文档，请不要在这里立即设置PDF文档的预览方式

  // 注册监听批注事件
  controller.registerAnnotationChangedListener((annotationChange: pdfViewManager.AnnotationChangedParam) => {
    if (annotationChange.controlType === 0) {
      // ADD-0, MOD-1, DEL-2
      for (let i = 0; i < annotationChange.pageIndexArray.length; i++) {
        manualAnnNumRecord[annotationChange.pageIndexArray[i]]++;
      }
      // 添加批注后自动退出批注模式，恢复正常浏览
      controller.disableAnnotation();
      AppStorage.setOrCreate('isAnnotationMode', false);
    }
  });

  // 监听选中的批注信息
  controller.registerAnnotationSelectedListener((annot: pdfViewManager.SelectedAnnotation | undefined) => {
    hilog.info(0x0000, 'testTag', 'annotation index %d, page index %d', annot?.annotationIndex, annot?.pageIndex);
  });
}
```

Two things here are worth copying verbatim: `registerPageCountChangedListener`
**before** `loadDocument`, and `disableAnnotation()` inside the ADD callback so a
single tap on the button produces exactly one annotation.

### Deleting every user-added annotation

`DelAllPdfAnnotation.zip#entry/src/main/ets/toolability/PdfKitAbility.ets`

```ts
export async function delAllAnn(store: relationalStore.RdbStore | null, controller: pdfViewManager.PdfController,
  manualAnnNumRecord: number[]): Promise<boolean> {
  let flag: boolean = false;
  if (store === null) {
    hilog.error(0x0000, 'testTag', 'store is null');
    return new Promise<boolean>((resolve) => {
      resolve(flag);
    });
  }

  // RdbPredicates是一次性对象
  let predicates = new relationalStore.RdbPredicates('ANNOTATION');
  let resultSet = await store.query(predicates, ['ID', 'pageIndex', 'rawAnnNumber', 'manualAnnNumber']);
  // resultSet是一个数据集合的游标，默认指向第-1个记录，有效的数据从0开始。
  while (resultSet.goToNextRow()) {
    const manualAnnNumber = resultSet.getLong(resultSet.getColumnIndex('manualAnnNumber'));
    if (manualAnnNumber !== 0) {
      flag = true;
      const id = resultSet.getLong(resultSet.getColumnIndex('ID'));
      const pageIndex = resultSet.getLong(resultSet.getColumnIndex('pageIndex'));
      const rawAnnNumber = resultSet.getLong(resultSet.getColumnIndex('rawAnnNumber'));
      // 循环删除标注
      for (let delCnt = 0; delCnt < manualAnnNumber; delCnt++) {
        controller.deleteSelectedAnnotation(rawAnnNumber, pageIndex);
      }
      // 更新数据库，使用唯一的key去更新数据
      let predicatesUpdate = new relationalStore.RdbPredicates('ANNOTATION');
      predicatesUpdate.equalTo('ID', id);
      const valueBucket: relationalStore.ValuesBucket = {
        'pageIndex': pageIndex,
        'rawAnnNumber': rawAnnNumber,
        'manualAnnNumber': 0,
      };
      let updateNum: number = await store.update(valueBucket, predicatesUpdate);
      // 重置manualAnnNumRecord
      manualAnnNumRecord[pageIndex] = 0;
    }
  }
  // 释放数据集的内存
  resultSet.close();
  return new Promise<boolean>((resolve) => {
    resolve(flag);
  });
}
```

This is the one function that gets the RDB hygiene right: a fresh
`RdbPredicates` for the update, keyed on the unique `ID` rather than on
`pageIndex`, and `resultSet.close()` at the end with a comment saying why. Copy
that discipline into the other three query sites (HW-05-0149, HW-05-0150).

What it gets wrong is the source of `manualAnnNumber`: it is read from the
database, which only the Save button writes (HW-05-0148). The corrected
condition merges the persisted count with the in-memory one:

```ts
const pending = manualAnnNumRecord[pageIndex] ?? 0;
const total = manualAnnNumber + pending;
if (total > 0) {
  flag = true;
  for (let delCnt = 0; delCnt < total; delCnt++) {
    controller.deleteSelectedAnnotation(rawAnnNumber, pageIndex);
  }
  // ... persist manualAnnNumber = 0 and reset manualAnnNumRecord[pageIndex] = 0
}
```

### Saving

`DelAllPdfAnnotation.zip#entry/src/main/ets/pages/Index.ets`

```ts
Button($r('app.string.save'))
  .height('28')
  .margin({ right: 15 })
  .onClick(async () => {
    // 退出批注模式，恢复正常浏览
    if (this.isAnnotationMode.get()) {
      this.controller.disableAnnotation();
      this.isAnnotationMode.set(false);
    }

    // 保存文件将覆盖源文档
    let result: number = await this.controller.saveDocument(this.filePath);
    if (result) {
      await saveAllAnnotation(this.store, this.manualAnnNumRecord, this.pageCount.get());
      this.getUIContext().getPromptAction().showToast({ message: '保存成功' });   // HW-05-0154
    } else {
      this.getUIContext().getPromptAction().showToast({ message: '保存失败' });
    }
  })
```

`PdfController.saveDocument` returns `Promise<number>` and the truthiness test is
correct - the official guide writes the same check as
`result ? 'success' : 'fail'` in `pdf-pdfview-open.md`.

### Delete-all button, including the re-save

`DelAllPdfAnnotation.zip#entry/src/main/ets/pages/Index.ets`

```ts
Button($r('app.string.deleteAllAnn'))
  .onClick(async () => {
    let result: boolean = await delAllAnn(this.store, this.controller, this.manualAnnNumRecord);
    if (result) {
      // 删除完所有批注，文件需自动保存，防止后续打开app出现上次标注而无法删除
      let saveResult = await this.controller.saveDocument(this.filePath);
      if (saveResult) {
        this.getUIContext().getPromptAction().showToast({ message: '已删除所有批注' });
      } else {
        this.getUIContext().getPromptAction().showToast({ message: '已删除所有批注，但保存文件失败' });
      }
    } else {
      this.getUIContext().getPromptAction().showToast({ message: '页面无批注可删除' });
    }
  })
```

The forced save after a deletion is the right call and the comment explains it:
without it, the annotations are gone from the view but still in the file, and on
the next launch the database says `manualAnnNumber = 0` for a document that still
contains them - they become undeletable.

### Entering annotation mode

`DelAllPdfAnnotation.zip#entry/src/main/ets/pages/Index.ets`

```ts
Button($r('app.string.addHighLightAnn'))
  .onClick(() => {
    // 进入批注模式
    this.controller.enableAnnotation(pdfViewManager.SupportedAnnotationType.HIGHLIGHT, 0xFFEEEAAA);
    this.isAnnotationMode.set(true);
  })
```

## Permissions & config

**No permissions at all.** `DelAllPdfAnnotation.zip#entry/src/main/module.json5`
has no `requestPermissions` array: the PDF ships inside the bundle, is copied
into `filesDir`, and both the temp copy and the database live in the app
sandbox. Nothing is read from or written to user storage, so nothing needs
`user_grant`.

`DelAllPdfAnnotation.zip#build-profile.json5`

```json5
"targetSdkVersion": "6.1.1(24)",
"compatibleSdkVersion": "6.1.1(24)",
"runtimeOS": "HarmonyOS",
"buildOption": {
  "strictMode": {
    "caseSensitiveCheck": true,
    "useNormalizedOHMUrl": true
  }
}
```

Required project asset - the sample cannot start without it, and the document
does not mention it (HW-05-0153):

```
entry/src/main/resources/rawfile/a.pdf
```

`module.json5` declares `deviceTypes: ["phone", "tablet", "2in1"]`, matching the
`PdfView` reference, which lists Phone, PC/2in1 and Tablet.

## Constraints

- **Versions.** DevEco Studio 6.1.1 Release or later, API Version 24 Release or
  later. `PdfView` itself is available **since 5.0.0(12)**.
- **`registerPageCountChangedListener` may be called only once, before
  `loadDocument`.** Both the sample
  (`PdfKitAbility.ets:115`) and the official reference say so - the `PdfView`
  reference example comments the call with "This listening method can be called
  only once before the document is loaded."
- **Do not set the preview mode immediately after `loadDocument`.** The official
  reference example carries the same warning ("Note: The document is just loaded.
  Do not set the preview mode of the PDF document immediately"), and the sample
  repeats it at `PdfKitAbility.ets:127`.
- **`getRawFileContentSync` takes a path relative to `resources/rawfile`.** The
  ResourceManager reference says the API "Obtains the content of a rawfile in the
  **resources/rawfile** directory" and its own example passes `"test.txt"`. The
  PDF Kit pages pass `'rawfile/input.pdf'`
  (`pdf-arkts-pdfview-component.md:74`, `pdf-open-document.md:50`), which
  prefixes the directory twice; the sample's `'a.pdf'` is the correct form.
- **Deletion is by index, and indices shift.** `deleteSelectedAnnotation(index,
  pageIndex)` called `N` times with the *same* index removes `N` consecutive
  annotations, because each removal moves the next one down. This only works
  because the user's annotations are appended after the original ones, so they
  occupy a contiguous range starting at `rawAnnNumber`.
- **The controller edits a temp copy.** Annotations live in
  `tempDir/temp<random>.pdf` until `saveDocument(filePath)` overwrites the
  original. The `temp${Math.random()}.pdf` naming comes straight from the
  official `pdf-open-document.md` sample; those temp files are not deleted by
  the app.
- **`controlType` is a raw number.** `0 = ADD`, `1 = MOD`, `2 = DEL`; there is no
  enum for it in the sample, only the `['ADD', 'MOD', 'DEL']` label array.
- **The reference pages for `pdfService` and `pdfViewManager` are stubs in this
  repository** (11 lines each), so the exact signatures of
  `deleteSelectedAnnotation`, `registerAnnotationChangedListener` and
  `registerAnnotationSelectedListener` could not be verified against the
  reference; they are documented here as the sample uses them.
- **Annotation type.** Only `SupportedAnnotationType.HIGHLIGHT` is wired up; the
  strikethrough variant is present but commented out at
  `PdfKitAbility.ets:130`.

## Pitfalls

- **Deciding what to delete from the database alone is incorrect**, because only
  the Save button writes that column. Add a highlight and press 删除所有批注
  without saving: `manualAnnNumber` is still 0 everywhere, nothing is deleted,
  and the user is told 页面无批注可删除 ("There are no annotations on the page to
  delete") while the highlight is on screen. Merge the in-memory
  `manualAnnNumRecord` into the count, or persist it as soon as the annotation
  is added. (HW-05-0148)
- **Leaving a `ResultSet` open is incorrect.** Four of the five query sites never
  close their cursor, including one inside the per-page loop of
  `saveAllAnnotation` that runs on every save. The reference is explicit: "If the
  resultSet is not closed, FD or memory leaks may occur." Close in a `finally`,
  as `delAllAnn` already does. (HW-05-0149)
- **Reusing one `RdbPredicates` across loop iterations is incorrect.**
  Conditions concatenate with AND by default, so the predicate becomes
  `pageIndex = 0 AND pageIndex = 1 AND ...` and matches nothing from the second
  page onwards, silently disabling the update branch. Build a fresh predicate per
  query - the sample's own comment says `RdbPredicates是一次性对象`
  ("RdbPredicates is a one-time object"). (HW-05-0150)
- **Never releasing the `pdfService.PdfDocument` is incorrect.** It is used once,
  in `aboutToAppear`, to count annotations, and then held open for the life of the
  page alongside the controller's own copy of the same file. Every official PDF
  Kit sample calls `releaseDocument()` once its work is done. (HW-05-0151)
- **`%d` and `%s` without `{public}` is incorrect.** The privacy identifier
  defaults to `{private}`, so the page indices and annotation counts that these
  traces exist to show print as `<private>`. (HW-05-0152)
- **The document says the byte stream is obtained through `@ohos.file.fs`, which
  is incorrect** - `context.resourceManager.getRawFileContentSync('a.pdf')`
  supplies the bytes and `@ohos.file.fs` only writes them into the sandbox. The
  document also never mentions the `resources/rawfile/a.pdf` asset the sample
  cannot run without, nor ResourceManager in 参考文档. (HW-05-0153)
- **Hardcoding the four result toasts in Chinese is incorrect** when every button
  label in the same file uses `$r('app.string.*')`; move them into
  `string.json`. (HW-05-0154)

## References

Reference and guide pages used to verify this card:

- `documentation/harmonyos-references/06_application-services/pdf-arkts-pdfview-component.md` -
  the `PdfView` parameters (`controller`, `pageLayout`, `isContinuous`,
  `showScroll`, `pageFit`), the "since 5.0.0(12)" version, the device list, and
  the two ordering rules quoted under Constraints.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/pdf-arkts-pdfview-component
- `documentation/harmonyos-guides/07_application-services/pdf-pdfview-open.md` -
  `PdfController.saveDocument(path, onProgress?): Promise<number>` and the
  `result ? 'success' : 'fail'` truthiness check.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/pdf-pdfview-open
- `documentation/harmonyos-guides/07_application-services/pdf-open-document.md` -
  the save-by-overwriting-the-source pattern with a `tempDir/temp<random>.pdf`
  copy, and the pdfService counterpart of `saveDocument`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/pdf-open-document
- `documentation/harmonyos-guides/07_application-services/pdf-pdfview-annotation.md` -
  `enableAnnotation(annotationType, color?)` and the annotation-mode flow.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/pdf-pdfview-annotation
- `documentation/harmonyos-guides/07_application-services/pdf-add-txt-img-annot.md`,
  `pdf-add-watermark.md`, `pdf-add-background.md`, `pdf-add-bookmark.md` - the
  `saveDocument` + `releaseDocument()` pairing on the pdfService side.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/pdf-add-txt-img-annot
- `documentation/harmonyos-guides/07_application-services/pdf-pdfview-switch-optimize.md` -
  `controller.releaseDocument()` before loading a different document.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/pdf-pdfview-switch-optimize
- `documentation/harmonyos-references/02_application-framework/arkts-apis-data-relationalstore-resultset.md` -
  `close()` and the FD/memory leak warning.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-data-relationalstore-resultset
- `documentation/harmonyos-references/02_application-framework/arkts-apis-data-relationalstore-rdbpredicates.md` -
  "Multiple predicates statements can be concatenated by using and() by default."
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-data-relationalstore-rdbpredicates
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` -
  `getRawFileContentSync(path)` and its `resources/rawfile`-relative path.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-references/03_system/js-apis-hilog.md` - the mandatory
  privacy identifier in `hilog` format strings.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
- The reference pages
  `documentation/harmonyos-references/06_application-services/pdf-arkts-pdfservice.md`
  and `pdf-arkts-pdfviewmanage.md` are 11-line stubs in this repository; the
  `PdfController` listener and annotation-deletion signatures could not be
  verified against them.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/pdf-arkts-pdfservice
