---
id: NEWS-22
title: Bookshelf with local import - DocumentViewPicker to the sandbox, then Reader Kit renders the book
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/22_bookshelf_demo.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/bookshelf_demo-0000002334312212
sample: huawei_industry_tree/11_news_reading/downloads/BookshelfDemo.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit", "@kit.ReaderKit"]
apis: ["picker.DocumentViewPicker", "picker.DocumentSelectOptions", "fs.openSync", "fs.statSync", "fs.readSync", "fs.writeSync", "fs.closeSync", "bookParser.getDefaultHandler", "readerCore.ReaderComponentController", ReadPageComponent, Navigation, NavDestination, NavPathStack, pushPathByName, "@Provide", "@Consume", Grid, bindMenu]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-11-0023, HW-11-0024, HW-11-0029, HW-11-0031]
status: verified-with-fixes
---

## When to use

Load this card when the app has to **take a file the user already owns and make
it the app's own** - a book, a document, a font, a backup - and then open it
with a component that needs a real sandbox path. That is two distinct steps
that are easy to conflate: the picker hands back a URI you may read once, and
everything afterwards has to work from a copy inside `filesDir`.

The pattern here is `DocumentViewPicker` filtered by suffix, an `fs` read into
an `ArrayBuffer`, a write into `context.filesDir`, and then Reader Kit's
`ReadPageComponent` pointed at the sandbox path. Around it sits an ordinary
shelf: grid/list toggle, multi-select edit mode with delete, a search page with
editable history, and a reading-history list.

**Read `HW-11-0023` before adopting the import routine.** The sample opens the
picker's URI `READ_WRITE`, which the platform does not grant, and swallows the
failure - so the shelf gains an empty book instead of the one the user chose.
It is the single most copy-pasted block in this sample (it appears twice,
verbatim) and the one you must not copy as-is.

## Feature checklist

- A shelf of book covers in a 3-column grid, with a `+` tile at the end.
- A settings menu in the title bar with three entries: 列表展示 / 九宫格展示
  (switch between list and grid), 导入本地书 (import a local book), 编辑书架
  (edit the shelf).
- Import opens the system document picker filtered to
  `.epub .txt .mobi .azw .azw3`, one file at a time.
- The chosen file is copied into the app sandbox and appears on the shelf under
  its decoded file name.
- Edit mode puts a checkbox on each cover, enables Delete and Collect, and
  shows a select-all row pinned near the bottom.
- Tapping a book pushes the reader page, which renders it through Reader Kit;
  the reader instance is released on leaving.
- The search page filters the shelf by substring, keeps a chip list of past
  queries, and has a delete mode that turns each chip into `word ×`.
- Opening a book from the shelf or from search records it in 阅读历史 (reading
  history), most recent first, without duplicates.

## Architecture

One `entry` module. A single `Navigation` at the root, three `NavDestination`
pages declared in `route_map.json`, and all shared state published from the
entry page with `@Provide`.

```
entry/src/main/ets
├── component/BookGridItem.ets     one cover in the grid + its edit-mode checkbox
├── component/BookListItem.ets     the list-mode row
├── component/Search.ets           the shelf's search bar - a TextInput that only navigates
├── component/TitleBar.ets         title + history icon + the three-item settings menu (holds a second copy of the import routine)
├── constants/CommonConstants.ets  every dimension, colour key and route name
├── entryability/EntryAbility.ets  full-screen window, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── model/Book.ets                 the Book interface + DisplayMode enum
├── model/HistoryWordModel.ets     HistoryWordModel + seven seeded search words
└── pages
    ├── MainPage.ets               @Entry: Navigation host, the grid/list, the + tile, delete
    ├── ReadHistory.ets            NavDestination: the history list
    ├── ReadPage.ets               NavDestination: Reader Kit
    └── SearchPage.ets             NavDestination: search, history chips, results
```

The documented tree matches the zip exactly.

**The design decision worth copying** is the routing shape. `MainPage` owns one
`NavPathStack` and publishes it as `@Provide('pathStack')`; every other page is
a `@Builder` function exported from its own file and registered in
`resources/base/profile/route_map.json`:

```json5
{ "name": "ReadPage", "pageSourceFile": "src/main/ets/pages/ReadPage.ets", "buildFunction": "ReadPageBuilder" }
```

Nothing imports another page. A component navigates by name and payload -
`this.pathStack.pushPathByName(CommonConstants.READ_PAGE, this.bookData.name)` -
and the route map resolves it, which also means the destination module is only
loaded when first visited. The shared data (`bookData`, `historyData`,
`selectedBooks`, `isEditMode`) travels the same way, as `@Provide` at the root
and `@Consume` wherever needed, so a grid item can append to the history list
without a callback chain.

**The structure worth avoiding** is the import routine itself. The identical
~35 lines appear in `MainPage.ets` (the `+` tile) and in `TitleBar.ets` (the
导入本地书 menu item), byte for byte, defect included. Extract it into a
`importBook(context): Promise<Book | undefined>` in a utils file before you
build on this - as it stands, `HW-11-0023` has to be fixed in two places.

## Implementation steps

1. **Filter the picker by suffix**, not by MIME: `DocumentSelectOptions` with
   `fileSuffixFilters = ['.epub', '.txt', '.mobi', '.azw', '.azw3']` and
   `maxSelectNumber = 1`.
2. **Bail out on an empty result** before touching `fs` - a cancelled picker
   resolves with an empty array, not a rejection.
3. **Open the returned URI `READ_ONLY`** (`HW-11-0023`). The guide is explicit:
   "The permission for the URI returned by `select()` of Picker is a temporary
   read-only permission."
4. **Size, allocate, read**: `fs.statSync(file.fd).size` into a new
   `ArrayBuffer`, then `fs.readSync(file.fd, arrayBuffer)`.
5. **Never leave the catch empty, and guard the close** (`HW-11-0023`). A
   failed `openSync` leaves `file` `undefined`, and `fs.closeSync(undefined)`
   in the `finally` throws over the top of the original error.
6. **Skip the destination write when the read failed**, otherwise the sandbox
   gains a zero-length file and the shelf a book that opens to nothing.
7. **Decode the file name** from the last URI segment with `decodeURI` before
   using it as a sandbox path - picker URIs are percent-encoded and Chinese
   book titles are the normal case here.
8. **Write into `context.filesDir`** with `CREATE | READ_WRITE`, and reuse that
   same `filesDir + '/' + name` when opening the book later.
9. **Dedupe search history with `filter`, never `splice` inside `forEach`**
   (`HW-11-0024`).
10. **Release the reader in `aboutToDisappear`** with
    `readerComponentController.releaseBook()`.

## Verified snippets

All snippets are from `BookshelfDemo.zip`. Corrected forms are marked.

**Import - `entry/src/main/ets/pages/MainPage.ets`** (corrected, see `HW-11-0023`)

```typescript
.onClick(async () => {
  let documentSelectOptions = new picker.DocumentSelectOptions();
  documentSelectOptions.maxSelectNumber = 1;
  documentSelectOptions.fileSuffixFilters = ['.epub', '.txt', '.mobi', '.azw', '.azw3'];
  let documentPicker = new picker.DocumentViewPicker();
  let documentSelectResult = await documentPicker.select(documentSelectOptions);
  if (!documentSelectResult || documentSelectResult.length <= 0) {
    return;                                        // cancelled: resolves empty, does not reject
  }
  // get book file path
  let srcPath: string = documentSelectResult[0];
  let file: fs.File | undefined = undefined;
  let arrayBuffer: ArrayBuffer | undefined = undefined;
  try {
    file = fs.openSync(srcPath, fs.OpenMode.READ_ONLY);   // FIX: shipped code asks for READ_WRITE
    let fileSize: number = 0;
    fileSize = fs.statSync(file.fd).size;
    arrayBuffer = new ArrayBuffer(fileSize);
    fs.readSync(file.fd, arrayBuffer);
  } catch (error) {
    hilog.error(0xFF00, 'Bookshelf', 'Failed to read, err: %{public}s', JSON.stringify(error));
  } finally {
    if (file) {                                    // FIX: shipped code calls closeSync(undefined)
      fs.closeSync(file);
    }
  }
  if (!arrayBuffer) {
    return;                                        // FIX: shipped code creates the book anyway
  }

  let segments = srcPath.split('/').filter(s => s.length > 0);
  let encodedFileName = segments[segments.length - 1];
  let fileName = decodeURI(encodedFileName);
  let destFile: fs.File | undefined = undefined;
  try {
    // 拷贝文件到沙箱
    destFile =
      fs.openSync(this.context.filesDir + `/${fileName}`, fs.OpenMode.CREATE | fs.OpenMode.READ_WRITE);
    fs.writeSync(destFile.fd, arrayBuffer);
  } catch (err) {
    hilog.info(0xFF00, 'Bookshelf', 'Failed to copy, err: %{public}s', JSON.stringify(err));
  } finally {
    if (destFile) {
      fs.closeSync(destFile);
    }
  }
  this.bookData.push({
    name: fileName,
    tag: CommonConstants.BOOK_TAG,
    imgUrl: $r('app.media.img1'),
    bookStatus: CommonConstants.BOOK_STATUS,
    readStatus: CommonConstants.READ_STATUS,
    latestReadTime: CommonConstants.READ_TIME
  });
})
```

**Three things about this block deserve attention.** First, the open mode: a
picker URI carries a *temporary read-only* grant, so `READ_WRITE` is refused -
and the shipped empty `catch` turns that refusal into silence. Second, the
`finally` in the shipped code is `fs.closeSync(file)` with `file` typed
`fs.File | undefined`; when `openSync` throws, that call receives `undefined`
and throws again, replacing the real error. Third, the two `try` blocks are
independent in the sample, so a failed read still reaches the write and still
pushes a `Book` - the shelf ends up with a titled cover over a zero-byte file,
and the failure only surfaces later inside Reader Kit.

The file name is recovered from the URI's last segment and `decodeURI`'d; the
same `fileName` is the key everything downstream uses - it is the sandbox path,
the shelf label, the navigation payload, and the identity used by delete and by
history dedupe. There is no id.

**Search history dedupe - `entry/src/main/ets/pages/SearchPage.ets`** (corrected, see `HW-11-0024`)

```typescript
dealHistoryData(wordModel: HistoryWordModel, isDelete: boolean) {
  if (wordModel.word.length !== 0) {
    // FIX: shipped code splices inside its own forEach, skipping the element after each removal
    this.historyWords = this.historyWords.filter(item => item.word !== wordModel.word);
    if (!isDelete) {
      this.historyWords.unshift(new HistoryWordModel(wordModel.word));
    }
  }
}
```

**One function, two jobs, and that is what makes it correct.** Searching a word
and deleting a chip both begin by removing every existing copy of it; only the
re-insert at the head is conditional. The shipped version instead walks the
array with `forEach` and `splice`s matches out of the array being walked, so
after a removal the next element shifts into the visited index and is skipped -
and the seeded history in `HistoryWordModel.ets` contains `'正确的学'` twice, so
the demo ships with the input that exposes it.

This exact function is copy-pasted from the `16_shopping` `SearchHistory`
sample (`HW-16-0010`), defect included. When you find it, fix it at the source
you copied from as well.

Note the neighbouring `addHistoryData` in the same file is written correctly -
`findIndex` then a single `splice` - which is a useful contrast: `splice` is
fine, iterating with `forEach` while splicing is not.

**Opening a book with Reader Kit - `entry/src/main/ets/pages/ReadPage.ets`** (as shipped)

```typescript
private readerComponentController: readerCore.ReaderComponentController = new readerCore.ReaderComponentController();
private readerSetting: readerCore.ReaderSetting = {
  fontName: '系统字体', fontPath: '', fontSize: 18, fontColor: '#000000', fontWeight: 400,
  lineHeight: 1.7, nightMode: true, themeColor: '#EAE2CF', themeBgImg: '', flipMode: '0',
  scaledDensity: display.getDefaultDisplaySync().scaledDensity > 0 ?
    display.getDefaultDisplaySync().scaledDensity : 1,
  viewPortWidth: 1216, viewPortHeight: 2490,
};
private bookParserHandler: bookParser.BookParserHandler | null = null;

aboutToAppear() {
  let fileName: string[] = this.pathStack.getParamByName('ReadPage') as string[];
  let filePath: string = this.context.filesDir + `/${fileName[0]}`;
  this.startPlay(filePath, '');
}

aboutToDisappear(): void {
  this.readerComponentController.releaseBook();      // must run, or the engine keeps the book open
}

private async startPlay(filePath: string, domPos: string) {
  try {
    let bookParserHandler = await bookParser.getDefaultHandler(filePath);
    let spineList: bookParser.SpineItem[] = bookParserHandler.getSpineList();
    let spineIndex: number = spineList[0].index;
    let context = this.getUIContext().getHostContext() as common.UIAbilityContext;
    let initPromise = await this.readerComponentController.init(context);
    let result: [bookParser.BookParserHandler, void] = await Promise.all([bookParserHandler, initPromise]);
    this.bookParserHandler = result[0];
    this.readerComponentController.registerBookParser(this.bookParserHandler);
    this.readerComponentController.setPageConfig(this.readerSetting);
    this.readerComponentController.startPlay(spineIndex || 0, domPos);
  } catch (err) {
    hilog.error(0x0000, 'testTag', 'startPlay: err: ' + JSON.stringify(err));
  }
}
```

**The order is the API contract**: parse the file into a handler, initialise the
controller against the ability context, register the handler, push the display
settings, and only then `startPlay` at a spine index. `getSpineList()[0]` is the
first chapter; `domPos` is the empty string for "start of chapter", and is where
a saved reading position would go on a resume.

`scaledDensity` is read from the default display with a `> 0` guard, and
`viewPortWidth`/`viewPortHeight` are the typesetting surface in px - these three
are what make the engine's line breaking match the device. `releaseBook()` in
`aboutToDisappear` is mandatory; the controller holds a native book instance
that outlives the component otherwise.

## Permissions & config

**None.** The sample declares no `requestPermissions` - and that is the point of
using a picker. `DocumentViewPicker` runs in the system's own file-manager UI,
so the app receives a grant for exactly the file the user chose without ever
holding a storage permission.

Two profile files carry the structure: `main_pages.json` lists only
`pages/MainPage`, and `route_map.json` declares `ReadPage`, `ReadHistory` and
`SearchPage` with their builder functions. Adding a page means adding a
`@Builder` export and one route-map entry - not an import.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`,
  matching the document. `deviceTypes` is `phone`, `tablet`, `2in1`.
- Reader Kit is a HarmonyOS system service; the sample sets `runtimeOS:
  "HarmonyOS"` and will not build against OpenHarmony.
- **The shelf is not persisted.** `bookData` is an in-memory array seeded empty;
  the imported file survives in `filesDir`, but the shelf entry does not survive
  a restart, and nothing scans `filesDir` on launch to rebuild it. Add a
  preferences or RDB layer before shipping.
- Every book gets the same placeholder cover (`app.media.img1`) and the same
  four constant strings for tag, status, read state and "last read". There is no
  metadata extraction, although `bookParser` could supply it.
- `deleteBooks()` removes the shelf entry only; the copied file stays in the
  sandbox forever.
- The select-all checkbox toggles its own icon and nothing else -
  `isFullSelected` is never read by the delete path, which works off
  `selectedBooks`. The Collect button has no handler at all.
- `BookGridItem` keeps its own `selected` boolean per item while the parent
  keeps `selectedBooks`; leaving edit mode does not reset either, so the ticks
  persist into the next edit session.
- The shelf's own `SearchBar` (`component/Search.ets`) is decorative - its
  `TextInput` has an `onClick` that pushes `SearchPage` and no `onChange`.
- `getParamByName('ReadPage')` returns every param pushed under that name;
  `fileName[0]` therefore reads the *oldest* one. Prefer
  `NavDestination.onReady`'s `context.pathInfo.param` if books can be opened
  repeatedly within one stack.

## Pitfalls

- **`HW-11-0023`** (B/high, confirmed): the picker's URI is opened with
  `fs.OpenMode.READ_WRITE` although the returned permission is documented as
  temporary read-only, so the open is denied; the empty `catch` swallows it,
  `arrayBuffer` stays `undefined`, and the code still creates the sandbox file
  and pushes a `Book` - the shelf gains an empty book. The `finally` also calls
  `fs.closeSync(file)` on an `undefined` handle. Fix: open `READ_ONLY`, log in
  the catch, return before creating the destination when the read failed, and
  guard the close with `if (file)`. Present twice: `MainPage.ets:84` and
  `TitleBar.ets` (the 导入本地书 menu item).
- **`HW-11-0024`** (B/low, confirmed): `dealHistoryData` splices
  `this.historyWords` inside its own `forEach`, so the entry after any removed
  one is skipped; the seeded history contains a duplicate word, so the demo
  reproduces it. The identical function is copy-pasted from the `16_shopping`
  `SearchHistory` sample (`HW-16-0010`). Fix: `filter` into a new array, then
  `unshift` - in both samples.

## References

- `documentation/harmonyos-guides/03_application-framework/select-user-file.md` - the picker's temporary read-only URI grant
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/select-user-file
- `documentation/harmonyos-references/02_application-framework/js-apis-file-picker.md` - `DocumentViewPicker`, `DocumentSelectOptions`, `fileSuffixFilters`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-picker
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `openSync` modes, `statSync`, `readSync`, `writeSync`, `closeSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/06_application-services/reader-api.md` - `bookParser`, `readerCore.ReaderComponentController`, `ReaderSetting`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/reader-api
- `documentation/harmonyos-references/06_application-services/reader-api-readpagecomponent.md` - `ReadPageComponent` and its callback
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/reader-api-readpagecomponent
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` - `NavPathStack`, `pushPathByName`, route maps
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `huawei_industry_tree/11_news_reading/docs/22_bookshelf_demo.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bookshelf_demo-0000002334312212
