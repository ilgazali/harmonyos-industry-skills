---
id: EDU-09
title: Import and open a PDF - DocumentViewPicker to PdfView, with the URI kept across pages
industry: 04_education
doc: huawei_industry_tree/04_education/docs/09_import_pdf.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/import_pdf-0000002378274393
sample: huawei_industry_tree/04_education/downloads/UploadPdf.zip
kits: ["@kit.CoreFileKit", "@kit.PDFKit", "@kit.AbilityKit", "@kit.ArkUI"]
apis: ["picker.DocumentViewPicker", "picker.DocumentSelectOptions", fileSuffixFilters, select, "fileIo.openSync", "fileIo.closeSync", "fs.File", PdfView, "pdfViewManager.PdfController", loadDocument, getPageCount, saveDocument, "pdfService.PageFit", Navigation, NavPathStack, NavDestination, NavDestinationContext, pathInfo, getParamByName, pushPathByName, Tabs, onContentWillChange, "@Provide", "@Consume", "@StorageProp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-04-0057, HW-04-0058, HW-04-0059, HW-04-0060, HW-04-0061, HW-04-0062, HW-04-0063, HW-04-0064, HW-04-0065, HW-04-0066, HW-04-0067, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card when a student needs to **bring their own PDF into the app** -
a past paper, a handout, a syllabus - and read it in place.

The one thing worth internalising: **this needs no permission at all.**
`DocumentViewPicker` is a system picker; the user's selection *is* the grant.
Reaching for `READ_WRITE_DOCUMENTS_DIRECTORY` or a media permission here is the
mistake the picker exists to prevent. Compare `EDU-08`, which ships PDFs as
rawfiles and copies them into the sandbox - use that when the documents are
yours, this when they are the user's.

The second thing is the shape of the plumbing: the picker returns a **URI**, and
that URI - not a name, not a path - is what every later page needs.

## Feature checklist

- A 导入文档 tile on the home page opens the system file picker.
- The picker is filtered to `.pdf`.
- The chosen document appears in a 最近文档 list with its file name.
- Tapping a recent document opens a full-page `PdfView`, fit to width.
- The empty-state artwork disappears once the first document is imported.
- Cancelling or failing the import leaves the app in a usable state.

## Architecture

One `entry` module, three pages, one small model file.

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── model/ListInfo.ets            ListInfo, HorizontalListInfo, HORIZONTALLISTITEM, RecentDocument
└── pages
    ├── ImportPdf.ets             @Entry - Navigation host, the tile row, the recent list
    ├── PrintPage.ets             document list with page counts
    └── PdfPreview.ets            the PdfView destination
```

The documented tree matches the zip.

**Three pieces of state are provided from the host and consumed downstream:**

| Provided in `ImportPdf` | Consumed by | Purpose |
| --- | --- | --- |
| `uriPrint: string[]` | `PrintPage` | every imported URI, in import order |
| `uriMap: Map<string, string>` | `PdfPreview` | file name → URI, for the preview lookup |
| `index: number` | `PrintPage` | the import counter |

**The map is the design mistake to avoid.** Keying by file name means the
navigation parameter can be a short display string - but file names are not
unique, so two documents called `试卷.pdf` from different folders collapse into
one entry (`HW-04-0062`). Carry the **URI** as the navigation parameter and keep
the name for display only; then no map is needed at all.

**`index` is worse.** It is the importer's running count in `ImportPdf` and
`PrintPage` then uses the same `@Consume` variable as its `for`-loop counter
(`HW-04-0066`), which both re-renders on every iteration and leaves the counter
holding the list length.

**The preview reads its parameter in `onReady`.** That is the right hook -
`NavDestinationContext` is where a destination first has its stack and its
parameter - but the sample reads it with `getParamByName`, which returns *every*
matching page's parameter rather than this one's (`HW-04-0058`).

## Implementation steps

1. **Create the picker with the ability context** and set the filter:
   `documentSelectOptions.fileSuffixFilters = ['.pdf']`. Declare no permission.
2. **Call `select` and handle both outcomes.** The promise rejects on failure;
   the sample attaches no `catch` (`HW-04-0063`).
3. **Keep the URI, not the name.** Open the file only to read `fs.File.name` for
   display, then close it, and store the URI as the identity.
4. **Guard the close.** `fs.closeSync(file?.fd)` throws when the open failed;
   null-check first (`HW-04-0057`).
5. **Advance counters only on success**, so the parallel arrays stay parallel
   (`HW-04-0061`).
6. **Pass the URI as the navigation parameter** and read it back with
   `context.pathInfo.param`, not `getParamByName`.
7. **In the destination's `onReady`, open the URI, `await loadDocument`, then
   close** - in that order. The sample closes first and drops the promise
   (`HW-04-0060`).
8. **Dispatch on an action id, not on a localized caption** (`HW-04-0064`).
9. **Do not veto tab switches.** `onContentWillChange` returning `false` kills
   three of the four tabs (`HW-04-0059`).

## Verified snippets

All snippets are from `UploadPdf.zip`. Corrected forms are marked.

**Import — `entry/src/main/ets/pages/ImportPdf.ets`** (corrected, see `HW-04-0057`, `HW-04-0061`, `HW-04-0062`, `HW-04-0063`)

```typescript
import { picker } from '@kit.CoreFileKit';
import { fileIo as fs } from '@kit.CoreFileKit';
import { common } from '@kit.AbilityKit';

private context = this.getUIContext().getHostContext() as common.UIAbilityContext;
private documentPicker: picker.DocumentViewPicker = new picker.DocumentViewPicker(this.context);
private documentSelectOptions: picker.DocumentSelectOptions = new picker.DocumentSelectOptions();

// FIX: sample dispatches on item.name === getStringSync($r('app.string.import_pdf').id)
.onClick(() => {
  if (item.action !== ListAction.IMPORT_PDF) {
    return;
  }
  this.documentSelectOptions.fileSuffixFilters = ['.pdf'];
  this.documentPicker.select(this.documentSelectOptions)
    .then((documentSelectResult: string[]) => {
      if (documentSelectResult.length === 0) {
        return;                                   // user cancelled
      }
      const uri = documentSelectResult[0];
      let file: fs.File | null = null;
      try {
        file = fs.openSync(uri);                  // opened only to read the display name
      } catch (e) {
        hilog.error(0x0000, 'ImportPdf', `openSync failed: ${e}`);   // FIX: sample catch is empty
        return;
      } finally {
        if (file !== null) {                      // FIX: PdfPreview omits this guard
          fs.closeSync(file.fd);
        }
      }
      this.uriPrint.push(uri);                    // FIX: sample pushes and increments before the open
      this.recentDocList.push(new RecentDocument(file.name, uri));   // FIX: sample keys by name only
      this.isVisible = false;                     // hide the empty-state artwork
      this.pageStack.pushPath({ name: 'PrintPage' });
    })
    .catch((err: BusinessError) => {              // FIX: absent in the sample
      hilog.error(0x0000, 'ImportPdf', `picker select failed: ${err.code} ${err.message}`);
    });
})
```

**`DocumentViewPicker` needs no permission, and that is the point.** The user
choosing a file in the system picker *is* the authorisation; the returned URI
carries it. `fileSuffixFilters` narrows what the picker will offer - note the
leading dot, `'.pdf'`.

**`select` returns an array, and cancellation resolves it empty.** The sample
indexes `[0]` unconditionally, so a cancelled picker feeds `undefined` into
`openSync`.

**Open only for the name, then close.** `fs.File.name` is the base name with its
extension - the only reason to open the file at import time. Everything after
that should address the document by URI.

**Preview — `entry/src/main/ets/pages/PdfPreview.ets`** (corrected, see `HW-04-0057`, `HW-04-0058`, `HW-04-0060`)

```typescript
import { pdfService, PdfView, pdfViewManager } from '@kit.PDFKit';
import { fileIo as fs } from '@kit.CoreFileKit';

@Component
struct PdfPreview {
  controller: pdfViewManager.PdfController = new pdfViewManager.PdfController();
  @State name: string = '';
  @State failed: boolean = false;

  build() {
    NavDestination() {
      Column() {
        PdfView({
          controller: this.controller,
          pageFit: pdfService.PageFit.FIT_WIDTH,
          showScroll: true
        })
      }.height('100%')
    }
    .title(this.name)
    .onReady(async (context: NavDestinationContext) => {
      const uri = context.pathInfo.param as string;    // FIX: sample uses
      // String(context.pathStack.getParamByName('PdfPreview')), which returns EVERY
      // PdfPreview page's parameter as an array
      let file: fs.File | null = null;
      try {
        file = fs.openSync(uri);
        await this.controller.loadDocument(file.path); // FIX: sample loads AFTER closing,
      } catch (e) {                                    //      and drops the promise
        this.failed = true;
        hilog.error(0x0000, 'PdfPreview', `open/load failed: ${e}`);
      } finally {
        if (file !== null) {                           // FIX: sample calls closeSync(file?.fd)
          fs.closeSync(file.fd);
        }
      }
    })
  }
}
```

**`fs.closeSync(file?.fd)` is the defect to internalise.** When `openSync`
throws, `file` is `null`, `file?.fd` is `undefined`, and `closeSync(undefined)`
raises a *second* error - from inside `finally`, which discards the first one and
propagates out of `onReady` where nothing catches it. Optional chaining looks
like a null guard and is not: it protects the property access, not the call.
`ImportPdf.ets` in the same project gets this right; `PdfPreview.ets` does not.

**`getParamByName` is plural by design.** The reference: *"Obtains the parameter
information of **all** NavDestination pages with the specified name"*, returning
`Array<unknown>`. `String(['a.pdf'])` happens to be `'a.pdf'`, which is why one
open document works and two do not. `NavDestinationContext.pathInfo.param` is
the accessor scoped to the page being built.

**Order matters: open, load, close.** The URI's authorisation is what the open
descriptor represents. Closing first and then loading by raw path relies on
ambient access to a location outside the app sandbox.

**Page counts — `entry/src/main/ets/pages/PrintPage.ets`** (corrected, see `HW-04-0066`, `HW-04-0067`)

```typescript
async pushList(): Promise<ListInfo[]> {
  const result: ListInfo[] = [];
  for (let i = 0; i < this.uriPrint.length; i++) {     // FIX: sample loops on the @Consume index
    let file: fs.File | null = null;
    try {
      file = fs.openSync(this.uriPrint[i]);
      await this.controller.loadDocument(file.path);
      const pageCount = this.controller.getPageCount(); // FIX: sample assigns to an @State field
      const dir = `${this.getUIContext().getHostContext()?.filesDir}/${file.name}`;
      await this.controller.saveDocument(dir);          // FIX: sample appends another '.pdf'
      result.push(new ListInfo(file.name, pageCount));
    } catch (e) {
      hilog.error(0x0000, 'PrintPage', `document ${i} failed: ${e}`);
    } finally {
      if (file !== null) {
        fs.closeSync(file.fd);
      }
    }
  }
  return result;
}
```

**Never loop on a state variable.** `this.index` is `@Consume`, so each `i++`
writes back to the `@Provide` and marks both pages dirty - N documents cause 2N
re-render requests during one list build, and the shared counter ends up holding
the list length instead of the import count. `this.pageCount` being `@State` has
the same effect for the same reason.

**`getPageCount()` needs a loaded document**, which is why `loadDocument` is
awaited here and not in `PdfPreview` - the same API, used correctly in one file
and not the other.

## Permissions & config

**None.** `entry/src/main/module.json5` declares no `requestPermissions` block,
and it should not: `DocumentViewPicker` grants access to exactly the file the
user picked, for as long as the app holds the URI.

Routing is by route map, with `PrintPage` and `PdfPreview` as `NavDestination`
builders exported from their page files.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Only the first selected file is used.** `select` returns an array;
  `documentSelectResult[0]` is all the sample reads, and
  `DocumentSelectOptions.maxSelectNumber` is never set.
- Documents are addressed by **file name** throughout, so duplicates collide
  (`HW-04-0062`).
- Nothing persists across launches: `uriPrint`, `uriMap` and `recentDocList` are
  in-memory `@Provide` state, so the 最近文档 list is empty on every start even
  though `PrintPage` copies the files into `filesDir`.
- The tile row's captions are hard-coded Chinese in `HORIZONTALLISTITEM`
  (`'导入文档'`) while the tap handler compares against a **localized** string
  resource - so the import tile stops working in any locale where the two differ
  (`HW-04-0064`).
- Three of the four bottom tabs are empty `TabContent` blocks, and unreachable
  in any case (`HW-04-0059`).
- 智能扫描, PDF工具, 扫描证件 and 提取文字 are tiles with no handler.

## Pitfalls

- **`HW-04-0057` — `fs.closeSync(file?.fd)` in a `finally` throws when the open
  failed,** replacing the real error and crashing out of `onReady`. Optional
  chaining is not a null guard for a call argument.
- **`HW-04-0058` — `getParamByName` returns every matching page's parameter.**
  Use `context.pathInfo.param`.
- **`HW-04-0059` — `onContentWillChange` returns `false`,** so three tabs are
  dead. Third occurrence of this exact defect in this industry.
- **`HW-04-0060` — the descriptor is closed before `loadDocument`, and the
  promise is discarded,** so a failed load shows a blank `PdfView` with no log.
- **`HW-04-0061` — `index++` and the `uriPrint` push happen before the open,**
  so one failed import desynchronises the parallel arrays permanently.
- **`HW-04-0062` — documents are keyed by file name,** so two files with the same
  name from different folders collapse into one map entry.
- **`HW-04-0063` — two empty `catch` blocks and a picker promise with no
  `catch`.** Every failure path in this feature is silent.
- **`HW-04-0064` — the import action is dispatched by string-comparing two
  localized captions,** one of which is hard-coded Chinese in the model.
- **`HW-04-0065` — the document's snippets hide the `onReady` wrapper** and omit
  the `pushPath` that a successful import performs.
- **`HW-04-0066` — `PrintPage` loops on the shared `@Consume index`,**
  re-rendering both pages once per iteration and clobbering the import counter.
- **`HW-04-0067` — saved copies get a doubled extension** (`试卷1.pdf.pdf`),
  because `fs.File.name` already includes the suffix.
- **Do not request a file permission for this.** The picker is the grant;
  `READ_WRITE_DOCUMENTS_DIRECTORY` is a restricted permission and is not needed
  to read a file the user chose.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-file-picker.md` - `DocumentViewPicker`, `DocumentSelectOptions`, `fileSuffixFilters`, `select`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-picker
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `openSync`, `closeSync`, `fs.File.name` and `fs.File.path`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/06_application-services/pdf-arkts-pdfviewmanage.md` - `PdfController`, `loadDocument`, `getPageCount`, `saveDocument`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/pdf-arkts-pdfviewmanage
- `documentation/harmonyos-references/06_application-services/pdf-arkts-pdfview-component.md` - `PdfView`, `pageFit`, `showScroll`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/pdf-arkts-pdfview-component
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `getParamByName` returning `Array<unknown>`, `NavDestinationContext.pathInfo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `onContentWillChange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-guides/03_application-framework/select-user-file.md` - the picker as the no-permission way to read user files
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/select-user-file
- `EDU-08` - the same `PdfView` preview, but over PDFs shipped as rawfiles, and the correct `loadResult` gating this page lacks
