---
id: UTIL-01
title: Print app skeleton - five HARs behind one Navigation, with routeMap-driven feature pages and a native print bridge
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/01_practice-tools-app-architecture-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tools-app-architecture-v1-0000002041326514
sample: huawei_industry_tree/15_utilities/downloads/LogTemp.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CameraKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PDFKit", "@kit.PerformanceAnalysisKit", "@kit.PreviewKit"]
apis: [abilityAccessCtrl, camera, common, commonType, deviceInfo, display, distributedDataObject, fileIo, filePreview, fileUri, fileuri, fs]
permissions: [ohos.permission.CAMERA, ohos.permission.WRITE_IMAGEVIDEO, ohos.permission.READ_MEDIA, ohos.permission.WRITE_MEDIA, ohos.permission.MEDIA_LOCATION, ohos.permission.PRINT, ohos.permission.DISTRIBUTED_DATASYNC]
min_api: 20
modules: [NativePrint (har), Basic (har), Mine (har), Print (har), Temp (har), entry (entry)]
findings: [HW-15-0001, HW-15-0010, HW-15-0011, HW-15-0012, HW-15-0013, HW-15-0014, HW-15-0015, HW-15-0016, HW-15-0039, HW-15-0100, HW-15-0101, HW-15-0103, HW-15-0104]
status: verified-with-fixes
---

## When to use

Load this card when you are **laying out a multi-module HarmonyOS app** whose
features are big enough to own their own pages, resources and native
dependencies - a print tool, a scanner suite, a document utility. It is the
utilities industry's flagship architecture template, so its module layout is
the thing most likely to be copied wholesale into a new project.

The shape is: one thin `entry` HAP that owns nothing but the tab skeleton and
the ability, three *feature* HARs (`Print`, `Temp`, `Mine`) that each declare
their own `routerMap`, and two *common* HARs - `Basic` for constants, logger
and breakpoints, `NativePrint` for the C++ print bridge. Entry never imports a
feature page; it imports a component per tab and pushes everything else by
name onto a `NavPathStack`.

Take the layout, the routeMap wiring and the continuation flow. **Do not take
the code inside it unmodified**: eight of the nine findings on this card are
defects in shipped feature code, including one HIGH - the only two working
items in the 模板 (template) tab throw on tap (`HW-15-0010`).

## Feature checklist

- A three-tab home - 打印 (print), 模板 (template), 我的 (mine) - inside a
  `Navigation`, with the tab bar moving to the side on `lg` screens.
- Status bar content colour follows the visible tab and the pushed page.
- Print tab: add/manage a printer, then document, photo and camera-scan cards.
- Document print: pick a file with `DocumentViewPicker`, list it with size and
  type icon, set copies and colour mode, preview it with Preview Kit, print it
  through the native bridge.
- Photo print: pick from the album, crop / rotate / adjust brightness and
  saturation, restore the original, save, print.
- Template tab: two categories of printable templates in a responsive grid;
  tapping one previews a PDF (in-app `PdfView`, or the system preview sheet).
- Continuation: an image being edited on the phone can be resumed on a tablet,
  carried by a distributed data object plus a distributed file copy.

## Architecture

Six modules in one project - one HAP, five HARs.

```
entry/                              type: entry   the only HAP
├── src/main/module.json5            all seven permissions, continuable: true
└── src/main/ets
    ├── entryability/EntryAbility.ets   breakpoints, avoid areas, status bar, continuation
    ├── entrybackupability/
    └── pages
        ├── Index.ets                   @Entry: Navigation > GridRow > Tabs (the only page)
        └── Tabs.ets                    a second TabBar implementation - never imported

features/
├── Print/  har @log/print  routerMap: 5 pages; deps @log/basic, @log/nativeprint, libprint.so
│     components/  PrintMainPage · PrintConnectPage · PrintDocxPage (699 lines: list, copies,
│                  print) · PrintSelectImgPage · PrintEditImagePage (523: crop/rotate/adjust)
│                  · PrintCameraPage (591, unfinished)
│     utils/ CropUtil, DecodeUtil, EncodeUtil, AdjustUtil, OpacityUtil
│     workers/ AdjustBrightnessWork, AdjustSaturationWork (off-thread pixels)
│     viewModel/ IconListViewModel, MessageItem, OptionViewModel, RegionItem
├── Temp/   har @log/temp   routerMap: PDFViewPage, ContentsListPage
│     TempMainPage (category grids) · ContentsListPage (two-tab shell) · TablesListPage
│     (tiles + tap handler) · PDFViewPage · tools/FilePreviewManager (rawfile -> filesDir)
└── Mine/   har @log/mine   MineMainPage.ets only

common/
├── Basic/       har @log/basic  Logger, BasicCommonConstants, StyleConstants, BreakpointType,
│                                ContentInfo (the continuation payload)
└── NativePrint/ har @log/nativeprint  the libnativeprint.so bridge: PrintManager (10 wrappers),
                 Print.ets (PrinterInfo/PageSize/PageRange/PrintJob/PrintOption/Margin),
                 SelectionModel (media + quality presets)
```

The documented tree matches the zip, with one caveat: the document labels
`entry/src/main/ets/pages/Tabs.ets` as the "主Tab页" (main tab page), but
nothing imports it - the real tab bar is `Index.ets`'s `BuildTabs` builder,
and `Tabs.ets` is a leftover second implementation.

**The design decision worth copying** is that each feature HAR carries its own
`routerMap` profile and exports a `@Builder` function per page, so `entry`
resolves pages *by name* and never has a compile-time reference to a feature
page:

```typescript
// features/Print/src/main/resources/base/profile/route_map.json
{ "name": "PrintEditImagePage",
  "pageSourceFile": "src/main/ets/components/PrintEditImagePage.ets",
  "buildFunction": "PrintEditImagePageBuilder" }
```

```typescript
// pushed from anywhere, including from another HAR
AppStorage.get<NavPathStack>('pathStack')?.pushPathByName('ContentsListPage', '模板详情');
```

That is what keeps the dependency graph a DAG: `entry` depends on the three
feature HARs, every feature HAR depends on `Basic`, `Print` additionally on
`NativePrint`, and no feature HAR depends on another. Adding a fourth feature
is a new HAR, a route map, and one `TabContent` - nothing else changes.

The **cost** of the same decision, and the reason to be careful with it, is
that the `NavPathStack` is published into `AppStorage` under `'pathStack'` and
read back with a cast at every push site. There is no type safety on the route
name, no check that the target HAR is even in the build, and a typo is a
silent no-op.

## Implementation steps

1. **Make `entry` thin.** One page, one ability. Everything a user can see
   belongs to a feature HAR.
2. **Give every feature HAR a `routerMap`** in `module.json5` plus an exported
   `@Builder` per page, and push by name through the `NavPathStack` you put in
   `AppStorage` once, from `Navigation`'s `onAppear`.
3. **Compute the breakpoint once, in the ability**, from `windowSizeChange`,
   and publish it plus the avoid areas (`topRectHeight`, `bottomRectHeight`,
   already converted to vp) to `AppStorage`; components read them with
   `@StorageLink` / `@StorageProp` and map through `BreakpointType`.
4. **Wrap the native library in one HAR** whose `Index.ets` re-exports the
   functions and the model classes. Pages import `@log/nativeprint`, never
   `libnativeprint.so`.
5. **Copy rawfile assets into `filesDir` before previewing them** - Preview
   Kit and `PdfView` only open sandbox paths - and address them with a path
   *relative to* `resources/rawfile`, with no `rawfile/` prefix
   (`HW-15-0010`).
6. **Define A4 as 8268 x 11692** hundredths of a millimetre, not 11692 square
   (`HW-15-0011`).
7. **Guard the print button on an empty file list** and keep the copy count in
   the item model, not in one page-level `@State` (`HW-15-0012`).
8. **Clone the `PixelMap` you intend to restore to** - `crop`/`rotate` mutate
   in place, so an aliased "saved" reference restores nothing (`HW-15-0013`).
9. **Order the extension-to-MIME branches longest-first** (`docx` before
   `doc`, `xlsx` before `xls`) or match exactly, and write `readLen` bytes per
   chunk from a buffer sized in bytes when copying a continuation file
   (`HW-15-0014`).
10. **Guard every `closeSync` in a `finally`** (`HW-15-0016`) and **key
    `ForEach` on a real id** such as the file path (`HW-15-0039`).
11. **Prune `module.json5` to the permissions you actually use** before
    shipping anything based on this template (`HW-15-0001`).

## Verified snippets

All snippets are from `LogTemp.zip`. Corrected forms are marked.

**The responsive skeleton — `entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
import { PrintMainPage } from '@log/print';
import { TempMainPage } from '@log/temp';
import { MineMainPage } from '@log/mine';
import { BasicCommonConstants, ImageInfo, Logger, StyleConstants } from '@log/basic';

@Entry
@Component
struct Index {
  @State currentTabIndex: number = 0;
  @StorageLink('breakPoint') breakPoint: string = BasicCommonConstants.BREAK_POINT_SM;
  @StorageLink('systemBarColor') @Watch('barChange') systemBarColor: string = '#000000';
  private controller: TabsController = new TabsController();
  pageStack: NavPathStack = new NavPathStack();

  build() {
    Navigation(this.pageStack) {
      GridRow({ breakpoints: { value: BasicCommonConstants.BREAK_POINTS_VALUE,
                               reference: BreakpointsReference.WindowSize } }) {
        GridCol({ span: { sm: BasicCommonConstants.COLUMN_SM, md: BasicCommonConstants.COLUMN_MD,
                          lg: BasicCommonConstants.COLUMN_LG } }) {
          Tabs({ index: this.currentTabIndex, controller: this.controller,
                 barPosition: this.breakPoint === BasicCommonConstants.BREAK_POINT_LG
                   ? BarPosition.Start : BarPosition.End }) {
            TabContent() { PrintMainPage(); }
              .tabBar(this.BuildTabs($r('app.media.print'), $r('app.media.print_selected'), '打印', 0));
            TabContent() { TempMainPage(); }
              .tabBar(this.BuildTabs($r('app.media.temp'), $r('app.media.temp_selected'), '模板', 1));
            TabContent() { MineMainPage(); }
              .tabBar(this.BuildTabs($r('app.media.mine'), $r('app.media.mine_selected'), '我的', 2));
          }
          .vertical(this.breakPoint === BasicCommonConstants.BREAK_POINT_LG)
          .scrollable(false)
          .onChange((index: number) => {
            this.currentTabIndex = index;
            this.systemBarColor = index === 1 ? '#000000' : '#FFFFFF';
          });
        };
      };
    }
    .hideTitleBar(true)
    .mode(NavigationMode.Stack)
    .onAppear(() => {
      AppStorage.setOrCreate('pathStack', this.pageStack);
    });
  }
}
```

**`Navigation` outside, `Tabs` inside, and one breakpoint string driving
both.** The stack has to wrap the tabs, not the other way round, or a pushed
detail page would appear inside a tab and keep the tab bar. `barPosition` and
`vertical` flip together off the same `breakPoint`: at `lg` the bar becomes a
left rail, at `sm`/`md` it stays at the bottom.

`AppStorage.setOrCreate('pathStack', this.pageStack)` in `onAppear` is the one
line that turns a local stack into the app's router - without it none of the
routeMap pushes from inside the feature HARs resolve. `@StorageLink` plus
`@Watch('barChange')` on `systemBarColor` is the smaller pattern beside it: the
tab index writes a colour into AppStorage and the watcher pushes it to the
window, while the ability does the same for pushed pages from
`navDestinationSwitch`.

**Previewing a bundled document — `features/Temp/src/main/ets/tools/FilePreviewManager.ets`** (corrected, see `HW-15-0010`)

```typescript
import { filePreview } from '@kit.PreviewKit';
import fileIo from '@ohos.file.fs';

filePreview(fileName: string, fileType: PreviewType, context: Context) {
  this.fileName = fileName;
  this.fileInfo = {
    title: this.fileName,
    // com.example.logtemp is the bundle name - replace it with your own
    uri: 'file://com.example.logtemp/data/storage/el2/base/haps/entry/files/' + this.fileName,
    mimeType: fileType
  };
  this.copyFile(context);   // filePreview only opens sandbox paths, so stage the asset first
  filePreview.openPreview(context, this.fileInfo).catch((err: BusinessError) => {
    Logger.error('testTag', `file preview error: ${err.code} ${err.message}`);
  });
}

copyFile(context: Context) {
  let filePath: string = context.filesDir + '/' + this.fileName;
  let content: Uint8Array = context.resourceManager.getRawFileContentSync(this.fileName);
  //                                                                     ^ FIX: no 'rawfile/' prefix
  let fdSand = fileIo.openSync(filePath,
    fileIo.OpenMode.WRITE_ONLY | fileIo.OpenMode.CREATE | fileIo.OpenMode.TRUNC);
  try {
    fileIo.writeSync(fdSand.fd, content.buffer);
  } finally {
    fileIo.closeSync(fdSand.fd);      // FIX: shipped code also drops the accessSync(filePath) that
  }                                   //      throws before the file has ever been created
}
```

**Two independent bugs sit on top of each other here.** `getRawFileContentSync`
takes a path *already relative to* `resources/rawfile`, so the shipped
`'rawfile/' + fileName` resolves to `rawfile/rawfile/...` and throws; and the
two whitepaper PDFs the template items reference are not in the zip at all -
`resources/rawfile` holds the tile PNGs, `1.txt` and a `docx`. `PDFViewPage`
repeats the same prefix at line 66. Neither call site has a `try`/`catch`, and
`readPDFInfo` runs from `aboutToAppear`, so the failure is an uncaught
exception rather than a message.

The shipped `copyFile` also calls `fileIo.accessSync(filePath)` on the
*destination* before creating it - a leftover that throws on first run. The
correct order is the one above: read the asset, open the sandbox path with
`CREATE | TRUNC`, write, close in a `finally`. And note the `uri`:
`openPreview` wants `file://<bundleName>/<sandbox path>`, spelled out here by
hand with the bundle name inline, where `fileUri.getUriFromPath(filePath)`
would build the same string without hardcoding it.

**Assembling a print job — `common/NativePrint/src/main/ets/Model/Print.ets` and `features/Print/.../PrintDocxPage.ets`** (corrected, see `HW-15-0011`, `HW-15-0012`, `HW-15-0016`)

```typescript
// Print.ets — the page size every print path in the app uses
export class PageSize implements testNapi.PageSize {
  public static readonly iso_a4: PageSize =
    new PageSize('iso_a4_210x297mm', '4', 8268, 11692, 0, 0, 0, 0);
  //                                     ^^^^ FIX: shipped value is 11692 - a square A4
}

// PrintDocxPage.ets
startPrint(): void {
  if (!this.currentPrint) {
    PrintUtil.showToast('请连接打印机', this.getUIContext());     // "connect a printer"
    (AppStorage.get('pathStack') as NavPathStack).pushPathByName('PrintConnectPage', null, true);
    return;
  }
  if (this.fileList.length === 0) {                              // FIX: absent - fileList[0] below
    PrintUtil.showToast('请先添加文档', this.getUIContext());
    return;
  }
  if (!(this.currentPrint.printerId === PrinterInfo.example.printerId)) {
    try {
      let info = this.fileList[0];
      this.getPrintParam(info);
      let result: number = printStartPrintJob(this.taskPrintJob!, this.taskPageSize!,
        this.taskPageRange!, this.taskMargin!, this.taskPrintOption!);
      console.log('', result);
    } catch (err) {
      Logger.error(`startPrintJob failed: ${err}`);               // FIX: try/finally with no catch
    } finally {
      if (this.file) {                                            // FIX: closeSync(undefined)
        fs.closeSync(this.file);
        this.file = undefined;                                    // FIX: stale handle double-closed
      }
    }
  } else {
    PrintUtil.showToast('未选择打印机', this.getUIContext());      // "no printer selected"
  }
}

getPrintParam(info: PrintFileModel): void {
  this.file = fs.openSync(info.path, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
  this.printFdList = [this.file.fd];
  this.taskPrintJob = new PrintJob(this.printFdList, 1, 600, this.currentPrint.printerId, 1, 0, 0, 0);
  this.taskPageSize = PageSize.iso_a4;
  this.taskPageRange = new PageRange(1, 1, [1]);
  this.taskMargin = new Margin(10, 10, 10, 10);
  this.taskPrintOption = new PrintOption('application/pdf',
    PrintUtil.removeSpaces(this.currentPrint.printerName), this.currentPrint.printerUrl,
    'application/pdf', SelectionModel.mediaTypeNormal.obj,
    SelectionModel.printQualityStandard.id.toString(), 0, '');
}
```

**The native bridge takes file descriptors, and that is what shapes this
code.** `PrintJob` carries an `fdList`, so the document has to be open at the
moment the job is submitted and closed after - which is why `this.file` is a
component field written by `getPrintParam` and closed by `startPrint`'s
`finally`. That split is exactly what makes the unguarded close dangerous: on
the "no document" path `getPrintParam` never ran, `this.file` is `undefined`,
and `fs.closeSync(undefined)` throws *inside the finally*, replacing the real
error with a cleanup error (`HW-15-0016`, which recurs in two other utilities
samples).

`PageSize.iso_a4` is a single static shared by the document, image and PDF
print paths, so the square-A4 typo distorts every job the app submits; the
units are hundredths of a millimetre, and 210 x 297 mm is 8268 x 11692.
`PrinterInfo.example.printerId` next to it is a sentinel for "nothing
selected" - a placeholder instance compared by id, where an `undefined`
printer would say the same thing without a magic object.

**Continuing an edit onto another device — `entry/src/main/ets/entryability/EntryAbility.ets`** (corrected, see `HW-15-0014`)

```typescript
async onContinue(wantParam: Record<string, Object | undefined>): Promise<AbilityConstant.OnContinueResult> {
  let sessionId: string = distributedDataObject.genSessionId();
  wantParam.distributedSessionId = sessionId;   // the peer joins this session
  // assets: one commonType.Asset per image (name/uri/path/size), built by getAssetInfo
  let contentInfo: ContentInfo = new ContentInfo(AppStorage.get('imageUriArray'), assets);
  this.distributedObject = distributedDataObject.create(this.context, contentInfo.flatAssets());
  this.distributedObject.setSessionId(sessionId);
  await this.distributedObject.save(wantParam.targetDevice as string);
  return AbilityConstant.OnContinueResult.AGREE;
}

// on the receiving device, once the object reports 'restored'
private fileCopy(attachment: commonType.Asset): void {
  let filePath: string = this.context.distributedFilesDir + '/' + attachment.name;
  let savePath: string = this.context.filesDir + '/' + attachment.name;
  let saveFile: fs.File | undefined = undefined;
  let file: fs.File | undefined = undefined;
  try {
    if (!fs.accessSync(filePath)) {
      return;
    }
    saveFile = fs.openSync(savePath, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
    file = fs.openSync(filePath, fs.OpenMode.READ_WRITE);
    let buf: ArrayBuffer = new ArrayBuffer(Number(attachment.size));  // FIX: shipped code x 1024
    let readSize = 0;
    let readLen = fs.readSync(file.fd, buf, { offset: readSize });
    while (readLen > 0) {
      readSize += readLen;
      fs.writeSync(saveFile.fd, buf, { length: readLen });            // FIX: shipped writes whole buf
      readLen = fs.readSync(file.fd, buf, { offset: readSize });
    }
  } catch (error) {
    hilog.error(0x0000, '[EntryAbility]', `file copy failed: ${JSON.stringify(error)}`);
  } finally {
    if (file) { fs.closeSync(file); }            // FIX: shipped opens before the try block and
    if (saveFile) { fs.closeSync(saveFile); }    //      closes unconditionally in finally
  }
}
```

**Continuation here is two channels, not one.** The distributed data object
carries the *structure* - a `ContentInfo` holding the image list and the asset
descriptors - while the image bytes travel as distributed-file assets that the
receiving device copies out of `distributedFilesDir` into its own `filesDir`.
`onCreate`/`onNewWant` both call `restoreDistributedObject`, which returns
immediately unless `launchReason` is `CONTINUATION`, then subscribes to
`'status'` before `setSessionId` - that order matters, because the restore
event can arrive as soon as the session is joined. The handler sets
`AppStorage['statusChange']`, and `Index`'s `onAppear` reads it to push
`PrintEditImagePage` with the restored `PixelMap`, so the user lands back in
the editor rather than on the home tab.

Two things in the shipped copy loop are wrong for the same reason: the code
treats `attachment.size` as a chunk hint rather than a byte count. It
allocates `size * 1024` (the asset size is already bytes, from
`fs.statSync(...).size` in `getAssetInfo`) and then writes the *whole* buffer
on every pass instead of the `readLen` bytes just read, so the persisted copy
is padded with whatever the tail of the buffer held.

## Permissions & config

```json5
// entry/src/main/module.json5
"continuable": true,                         // enables onContinue / CONTINUATION launch
"requestPermissions": [
  { "name": "ohos.permission.READ_MEDIA",           ... "when": "always" },   // deprecated
  { "name": "ohos.permission.WRITE_MEDIA",          ... "when": "always" },   // deprecated
  { "name": "ohos.permission.MEDIA_LOCATION",       ... "when": "always" },
  { "name": "ohos.permission.PRINT" },                                        // no reason/usedScene
  { "name": "ohos.permission.DISTRIBUTED_DATASYNC", ... "when": "always" },   // unused
  { "name": "ohos.permission.CAMERA",               ... "when": "always" },
  { "name": "ohos.permission.WRITE_IMAGEVIDEO",     ... "when": "always" }    // restricted
]
```

- **Prune this block before reusing it** (`HW-15-0001`). Full-source greps find
  no distributed-datasync API use and no album write that needs
  `WRITE_IMAGEVIDEO`; `READ_MEDIA`/`WRITE_MEDIA` are the pre-API-12 media
  permissions and are superseded by the picker-based flows the sample actually
  uses.
- `when: "always"` is declared for every permission including `CAMERA`, which
  only runs while the camera page is open - `"inuse"` is the honest value.
- `features/Print/src/main/module.json5` re-declares `CAMERA` and
  `WRITE_IMAGEVIDEO` at HAR level; declarations merge at build time, so the
  entry copies are redundant.
- The `Print` and `Temp` HARs each declare `"routerMap": "$profile:route_map"`;
  that profile is what makes `pushPathByName` resolve across module
  boundaries.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is
  `6.0.0(20)`. `deviceTypes` are phone, tablet and 2in1.
- **This is a framework sample, not an app.** The document says so plainly:
  template content is local data, to be replaced with a service. The 我的 tab
  is a static list, and both whitepaper PDFs are missing from the zip.
- **PDF preview and file preview need a physical device.** Both source files
  carry the comment 需要使用真机，模拟器不支持 (real device required, the
  emulator does not support it), and `TablesListPage` checks
  `deviceinfo.productModel == 'emulator'` and toasts instead.
- Ten of the twelve template tiles are stubs: the tap handler only acts on
  indexes 6 and 7 and toasts 功能暂未上线 (not yet available) for the rest -
  and those two are the ones broken by `HW-15-0010`.
- The native print bridge (`libnativeprint.so`) is a prebuilt C++ module; the
  ArkTS side is only wrappers, so printing needs real hardware.
- The bundle name `com.example.logtemp` is hardcoded into preview URIs in two
  files; renaming the bundle breaks preview silently.

## Pitfalls

- **`HW-15-0010`** (B/high, confirmed): **the whitepaper preview is doubly
  broken.** `getRawFileContentSync('rawfile/' + fileName)` resolves to
  `rawfile/rawfile/...` (paths are already relative to `resources/rawfile`),
  and the referenced PDFs are not shipped. The sync call throws with no
  `try`/`catch` in `readPDFInfo` or `filePreview`, so the template tab's only
  two working items die on tap. Fix: drop the prefix and ship the PDFs, or
  stub the items.
- **`HW-15-0011`** (B/medium, confirmed): **the default A4 page size is
  square** - `new PageSize('iso_a4_210x297mm', '4', 11692, 11692, ...)`, where
  the width should be ~8268. Every print path uses `PageSize.iso_a4`, so every
  job is submitted with a wrong paper width. Fix: correct the width constant.
- **`HW-15-0012`** (B/medium, confirmed): **立即打印 (print now) with no
  document crashes.** `startPrint` reads `this.fileList[0]` unguarded and
  `getPrintParam(undefined)` dereferences `info.path`; the `try`/`finally` has
  no `catch`, so it escapes. Separately, `printCount` is one page-level
  `@State` shared by every item's stepper and `TextInput`, so changing one
  item's copies changes every displayed count and writes the shared value into
  the touched item. Fix: guard `fileList.length`; move the count into the item
  model.
- **`HW-15-0013`** (B/medium, confirmed): **restore-original in the image
  editor is a guaranteed no-op.** `onReady` stores the same reference into
  both `pixelMap` and `savepixMap`; `crop()`/`rotate()` mutate in place, so the
  还原 button restores the already-mutated image. Fix: clone into `savepixMap`
  at load.
- **`HW-15-0014`** (B/low, confirmed): **MIME branch order makes the docx and
  xlsx branches unreachable** (`'docx'.includes('doc')` is true, so docx maps
  to `application/msword`), and the continuation copy allocates
  `attachment.size * 1024` and writes the full buffer per iteration instead of
  `readLen` bytes. Fix: match the extension exactly (or order longest-first)
  and write `{ length: readLen }`.
- **`HW-15-0015`** (B/low, confirmed): **the print camera page ships
  unfinished.** `switchFrontOrBackCamera()` has an empty body but is wired to
  the bottom-bar button; the `photoAvailable` callback decodes the JPEG buffer
  as raw `RGBA_8888` at a fixed size, `startCamera` then overwrites
  `finalPixelMap` with the last preview frame, and neither `finalPixelMap` nor
  `hasPicture` is used in `build()`; `release()` skips `previewOutput2`. Fix:
  finish the page or remove the stubs.
- **`HW-15-0016`** (B/low, confirmed): **systematic - `closeSync` in a
  `finally` on a possibly-undefined file.** `PrintDocxPage.startPrint` closes
  `this.file`, which is only assigned inside `getPrintParam`, so early-throw
  paths close `undefined` and mask the original error while a stale handle is
  double-closed later. Also in `Base64ImageSaveH5` and `PosterGen`. Fix:
  `if (file) { closeSync(file); }` and clear the field after closing.
- **`HW-15-0039`** (B/medium, confirmed): **systematic - `(item: string) => item`
  key generators over object arrays.** `PrintDocxPage.ets:623` keys a
  `ForEach` over `PrintFileModel[]` by stringifying the object, so every row
  gets the same key and the per-item delete removes the wrong row. Anchored on
  `UTIL-16`. Fix: return `item.path` or a real id.
- **`HW-15-0001`** (D/medium, confirmed): **the permission block ships an
  unused, a restricted and two deprecated permissions** - and as the industry's
  flagship template it is the block most likely to be copied verbatim. Fix:
  declare only what the code exercises.

## References

- `documentation/harmonyos-guides/01_getting-started/har-package.md` - HAR structure, `Index.ets` entry points, cross-module dependencies
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/har-package
- `documentation/harmonyos-guides/04_application-services/preview-kit-guide.md` - `filePreview.openPreview`, `PreviewInfo`, the sandbox-path requirement
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/preview-kit-guide
- `documentation/harmonyos-guides/04_system/native-print-file.md` - the native print interface behind `libnativeprint.so`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/native-print-file
- `documentation/harmonyos-guides/04_system/print.md` - print jobs, page sizes and printer discovery
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/print
- `documentation/harmonyos-guides/03_application-framework/data-sync-of-distributed-data-object.md` - `distributedDataObject`, session ids, the `'status'` / `'restored'` callback
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/data-sync-of-distributed-data-object
- `documentation/harmonyos-guides/03_application-framework/uiability-cross-device-interaction.md` - `continuable`, `onContinue`, `CONTINUATION` launch reason
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/uiability-cross-device-interaction
- `documentation/harmonyos-guides/05_media/image-kit.md` - `PixelMap`, `crop`, `rotate`, encoding
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-kit
- `documentation/harmonyos-guides/05_media/medialibrary-kit.md` - album access and the current media permission model
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/medialibrary-kit
- `UTIL-02` - the key-scenario index that lists the small samples built on this skeleton
- `UTIL-16` - the other instance of the `ForEach` key defect (`HW-15-0039`)
