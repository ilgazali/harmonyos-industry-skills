---
id: NEWS-25
title: Auto page turn - drive Reader Kit's flipPage from a setInterval the page owns
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/25_automatic_page_turn.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/automatic_page_turn-0000002415970301
sample: huawei_industry_tree/11_news_reading/downloads/AutomaticPageTurning.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit", "@kit.ReaderKit"]
apis: [bookParser, common, deviceInfo, display, fileIo, hash, hilog, readerCore, util, window]
permissions: [ohos.permission.LOCATION, ohos.permission.APPROXIMATELY_LOCATION, ohos.permission.LOCATION_IN_BACKGROUND]
min_api: 20
modules: [entry (entry)]
findings: [HW-11-0026, HW-11-0027, HW-11-0031]
status: verified-with-fixes
---

## When to use

Load this card when a reader page must **turn its own pages on a timer** -
hands-free novel reading, an accessibility mode, a kiosk or a demo loop. The
pattern is one line of substance: a `setInterval` whose callback calls
`ReaderComponentController.flipPage(true)`, plus the discipline of owning that
interval id properly.

The interesting part is not the timer, it is the state machine around it. An
auto-turning reader has three ways in and three ways out - the user taps the
菜单 (menu) to open settings, taps the page to reveal the menu bar, or leaves
the page entirely - and every one of them has to answer "is the interval
running right now?". This sample wires start/stop into four separate
handlers, and the one it forgets is the destructive one.

It generalises to anything a page drives periodically while it is visible:
slideshow advance, live-score polling, a rotating banner. The rule this card
exists to teach is that **a timer started by a page is owned by that page's
lifecycle**, and the counterpart of `startAutoScroll()` in `aboutToDisappear`
is not optional (`HW-11-0026`).

## Feature checklist

- A reader page renders a local `.txt` through `ReadPageComponent` and opens
  it at the first chapter.
- Tapping anywhere on the page reveals a three-button menu bar (catalog,
  collect, auto-turn).
- Tapping 自动翻页 (auto page turn) starts the interval immediately and
  switches the page into auto-turn mode.
- Tapping the page again while auto-turning stops the interval and raises a
  settings sheet.
- The sheet offers a page-turn *mode* (simulated page curl / horizontal
  slide) and a *speed* slider from 1 to 10 seconds per page.
- Dismissing the sheet, or tapping the dimmed background, resumes turning at
  the newly chosen speed.
- 结束自动翻页 (end auto page turn) stops the interval and returns to the
  normal reader.
- Leaving the page stops the interval before the book is released.
- Switching the system to dark mode re-tints the reader through
  `setPageConfig`.

## Architecture

One `entry` module. The whole feature lives in a single page; everything else
is Reader Kit bootstrap and window plumbing.

```
entry/src/main/ets
├── common
│   ├── BookParserInfo.ets       book metadata bean + LocalBookImportResult enum
│   ├── FontFileInfo.ets         alias/path pair for a custom font
│   └── LocalBookImporter.ets    hash, size-check and cache-copy an imported book
├── entryability
│   ├── EntryAbility.ets         loads pages/Reader, forces light mode, calls WindowAbility
│   └── WindowAbility.ets        singleton: avoid areas, window size, orientation -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages
│   └── Reader.ets               the entire feature: reader bootstrap, menu, sheet, the interval
└── utils
    ├── BookUtils.ets            suffix <-> BOOK_FILE_TYPE mapping, cache paths
    └── FileUtils.ets            exists / copy / mkdir -p helpers
```

The documented 工程目录 matches the zip file for file. What it does not tell
you is that **`common/` and `utils/` are dead in this sample**: `Reader.ets`
imports none of them. It reads a single `test.txt` out of `rawfile` and copies
it to `filesDir` in eleven lines of its own. The import pipeline, the sha256
hashing, the 300 MB threshold and the five supported book formats are all
carried over from the bookshelf sample (`NEWS-22`) - as are the unused string
resources `import_book`, `import_successful_tips`, `continue_reading` and
`go_to_reader`, and as are the three location permissions (`HW-11-0027`).

**The design decision worth avoiding** is exactly that copy-paste inheritance.
The reader page is 435 lines and self-sufficient; the 550 lines of import
machinery beside it are a template's residue, not a dependency. When you lift
this pattern, take `Reader.ets` and leave the rest - and read `module.json5`
before you keep it, because the residue there is three user-grant privacy
permissions for a sample that never resolves a location.

**The decision worth copying** is that `timerId` is a plain private field, not
`@State`. Nothing in the UI renders it, so making it observable would only
cost re-renders. The observable state is `scrollSpeed` (drives the slider
label), `currentIndex` (which menu is showing) and `isShow` (the sheet) -
three flags for what the user can see, one plain number for what the page owns.

## Implementation steps

1. **Copy the book out of `rawfile` into the sandbox.** Reader Kit's parser
   takes a file path, and a raw-file resource has none. `getRawFileContentSync`
   plus `fileIo.writeSync` into `context.filesDir` is the whole step.
2. **Bootstrap the controller and the parser in parallel.** `init(context)` and
   `bookParser.getDefaultHandler(path)` are both promises; `Promise.all` on the
   pair is what the reader guide prescribes. Register the parser, push the
   settings, then `startPlay`.
3. **Keep one `ReaderSetting` object** and mutate the field you want before
   calling `setPageConfig` again - that is how both the dark-mode switch and
   the `flipMode` radios apply their change.
4. **Start the interval with the speed in seconds x 1000**, and make
   `startAutoScroll` call `stopAutoScroll` first so a speed change cannot leave
   two intervals racing.
5. **Guard `stopAutoScroll` on a truthy id and zero it after clearing.** `0` is
   never a valid timer id, so it doubles as "not running".
6. **Stop the interval in `aboutToDisappear`, before `releaseBook()`**
   (`HW-11-0026`). Without it the interval outlives the page and keeps calling
   `flipPage` on a released controller.
7. **Pair start and stop at every UI edge**: the page-body tap stops,
   the sheet's `onWillDismiss` restarts, the menu button starts, the 结束
   button stops.
8. **Delete the location permissions from `module.json5`** (`HW-11-0027`). A
   reader that declares `LOCATION_IN_BACKGROUND` will not survive app review.

## Verified snippets

All snippets are from `AutomaticPageTurning.zip`. Corrected forms are marked.

**The timer pair — `entry/src/main/ets/pages/Reader.ets`** (as shipped)

```typescript
@State scrollSpeed: number = 3; // 默认速度3秒/页
// 定时器ID
private timerId: number = 0;

// 开始自动翻页
private startAutoScroll(): void {
  this.stopAutoScroll();
  this.timerId = setInterval(() => {
    this.readerComponentController.flipPage(true);
  }, this.scrollSpeed * 1000);
}

// 停止自动翻页
private stopAutoScroll(): void {
  if (this.timerId) {
    clearInterval(this.timerId);
    this.timerId = 0;
  }
}
```

**Three lines carry the design.** `stopAutoScroll()` as the first statement of
`startAutoScroll` makes start idempotent: the sheet's dismiss handler calls it
on every close, and a user who drags the slider three times still ends up with
exactly one interval. `this.timerId = 0` after `clearInterval` is what makes
the `if (this.timerId)` guard meaningful - the id becomes the running flag, so
no second boolean has to be kept in step. And `flipPage(true)` takes the
direction, not a page number, which is why the interval body needs no state of
its own.

Note what is *not* here: nothing checks whether the book has ended. `flipPage`
on the last page is a no-op, so the interval keeps firing forever at the end of
the novel. In a real reader, listen for the reader's page events and stop when
the last spine item is shown.

**The lifecycle hook — same file** (corrected, see `HW-11-0026`)

```typescript
aboutToDisappear(): void {
  this.stopAutoScroll();                 // FIX: absent in the sample
  display.off('change', this.screenDensityCallBack);
  this.readerComponentController.off('pageShow');
  this.readerComponentController.off('resourceRequest');
  this.readerComponentController.releaseBook();
}
```

**This is the defect the card exists for.** As shipped, leaving the reader
while auto-turn is active releases the book and leaves the interval running,
so every `scrollSpeed` seconds for the rest of the process lifetime a
`flipPage(true)` lands on a released controller. It is a leaked timer *and* a
use-after-release, and it is invisible in a demo because the demo never leaves
the page. The stop must come before `releaseBook()`, not after - otherwise a
callback already queued can still reach the released controller.

Two more things in this teardown are worth knowing before you copy it:
`screenDensityCallBack` is declared `null` and never assigned, and
`on('pageShow')` / `on('resourceRequest')` are never registered anywhere in the
sample - so three of these four lines unregister listeners that do not exist.
The reader guide's version registers `pageShow` in a `registerListener()` call
that this sample dropped. Keep the `off` calls only for the events you actually
subscribe to.

**The reader bootstrap — same file** (as shipped)

```typescript
private readerComponentController: readerCore.ReaderComponentController =
  new readerCore.ReaderComponentController();

aboutToAppear(): void {
  let filePath =
    this.copyRawfileToSanBox(this.getUIContext().getHostContext() as common.UIAbilityContext, 'test.txt');
  this.startPlay(filePath, 0, '');
}

private async startPlay(path: string, resourceIndex: number, domPos: string) {
  try {
    let context = this.getUIContext().getHostContext() as common.UIAbilityContext;
    let initPromise: Promise<void> = this.readerComponentController.init(context);
    let defaultHandler: Promise<bookParser.BookParserHandler> = bookParser.getDefaultHandler(path);
    let result: [bookParser.BookParserHandler, void] = await Promise.all([defaultHandler, initPromise]);
    this.defaultHandler = result[0];
    this.readerComponentController.registerBookParser(this.defaultHandler);
    this.readerComponentController.setPageConfig(this.readerSetting);
    this.readerComponentController.startPlay(resourceIndex || 0, domPos);
  } catch (err) {
    hilog.error(0x0000, TAG, 'startPlay: err: ' + JSON.stringify(err));
  }
}
```

**The ordering is the API contract, not a style choice.** `init` must run
before any other controller call; `registerBookParser` must run before
`startPlay`. The `Promise.all` overlaps the two independent waits - engine init
and file parse - into one, which is the difference between a reader that opens
in one beat and one that opens in two. The typed tuple
`[bookParser.BookParserHandler, void]` is required: ArkTS will not infer the
heterogeneous result of `Promise.all`.

`readerSetting` is built as a field initializer and reads
`viewPortWidth: this.windowWidth` from an `@StorageLink` that `WindowAbility`
fills asynchronously from `window.getLastWindow`. If the callback has not run
when the struct is constructed, the viewport is set from `0`. Seed the two
dimensions synchronously (`getMainWindowSync().getWindowProperties()`) if you
adopt this bootstrap.

**Start and stop across the UI edges — same file** (as shipped)

```typescript
.bindSheet($$this.isShow, this.buildAutoFlipPageSetting(), {
  height: '37%',
  onWillDismiss: ((dismissSheetAction: DismissSheetAction) => {
    this.isShow = false;
    if (this.currentIndex === 2) {
      this.startAutoScroll();          // resume with the newly picked speed
      this.showModalBanner = false;
    } else {
      this.closeModal();
    }
    dismissSheetAction.dismiss();
  }),
});

// the 自动翻页 menu button
.onClick(() => {
  this.currentIndex = 2;
  this.startAutoScroll();
  this.showModalBanner = false;
  this.isShow = false;
});

// the whole page
}.width('100%').height('100%').onClick(() => {
  this.showModal();                    // isShow = (currentIndex === 2)
  if (this.currentIndex === 2) {
    this.stopAutoScroll();             // pause while the settings sheet is up
  }
});
```

**`currentIndex === 2` is the auto-turn mode flag** and it appears in six
places: it gates the sheet content's `visibility`, it hides the menu bar, it
dims the background, and it decides in all three handlers above whether a tap
means "pause and configure" or "close the menu". A named boolean
(`autoFlipping`) would read better than a magic index, but the mechanism is
sound: **every entry to the sheet stops the timer and every exit restarts it**,
so the speed slider always takes effect on the next resume rather than
mid-interval.

`$$this.isShow` is the two-way binding that lets a swipe-down dismissal write
`false` back into the page's state. `onWillDismiss` then has to call
`dismissSheetAction.dismiss()` explicitly - registering the callback overrides
the default dismissal, so omitting that line makes the sheet impossible to
close.

**The speed control — same file** (as shipped)

```typescript
Row() {
  Text('翻页速度:')
    .fontSize(14);

  Slider({
    value: this.scrollSpeed,
    min: 1,
    max: 10,
    step: 1,
    style: SliderStyle.OutSet
  })
    .onChange((value: number) => {
      this.scrollSpeed = value;
    })
    .width('65%');

  Text(`${this.scrollSpeed}秒/页`)          // "N seconds per page"
    .fontSize(14);
}
```

`step: 1` with `min: 1` is what keeps `scrollSpeed * 1000` an integer number of
milliseconds and stops the label showing `3.4秒/页`. Changing the slider does
**not** restart the interval by itself - the new value is only read the next
time `startAutoScroll` runs, which is when the sheet closes. That is the
intended behaviour here, and it is why start being idempotent matters.

The two radios above the slider write `flipMode` `'0'` (仿真翻页, simulated
page curl) or `'1'` (横向滑动, horizontal slide) into the shared
`readerSetting` and re-push it with `setPageConfig`. Both radios share
`group: 'radioGroup'` but hardcode `.checked(true)` / `.checked(false)`
instead of binding to state, so after the user picks the second mode and
reopens the sheet, the first radio shows selected again while `flipMode` is
still `'1'`. Bind `checked` to `this.readerSetting.flipMode === '0'` if you
reuse this sheet.

## Permissions & config

Declared in `entry/src/main/module.json5` — **all three should be deleted**
(`HW-11-0027`):

```json5
"requestPermissions": [
  { "name": "ohos.permission.LOCATION",               /* delete */
    "reason": "$string:EntryAbility_desc",
    "usedScene": { "abilities": ["EntryAbility"] } },
  { "name": "ohos.permission.APPROXIMATELY_LOCATION", /* delete */ },
  { "name": "ohos.permission.LOCATION_IN_BACKGROUND", /* delete */ }
]
```

Corrected form: no `requestPermissions` block at all. The sample contains no
`geoLocationManager` import and no location call of any kind. Two further
signs that this block was never reviewed: the `reason` points at
`EntryAbility_desc` (the ability description, not a location justification),
and `usedScene` carries no `when`, which for `LOCATION_IN_BACKGROUND` is the
declaration most likely to be rejected outright.

`deviceTypes` is `["phone"]` only, although `EntryAbility` branches on
`deviceInfo.deviceType === 'tablet'` for orientation and `WindowAbility` also
handles `'2in1'` - more inherited code that this module cannot reach.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **Timers are suspended in the background.** The timer reference is explicit:
  "If the UI is switched to the background, the timer will be suspended", and
  expired timers fire in sequence once the app returns to the foreground. An
  auto-turning reader that is backgrounded for a minute will therefore not
  catch up page by page - which is what you want here, but do not build a
  reading-progress counter on this interval.
- The book is a single `test.txt` in `rawfile`, re-copied to `filesDir` on
  every `aboutToAppear` without an existence check.
- There is no end-of-book detection: `flipPage(true)` past the last page does
  nothing and the interval keeps running.
- The colour-mode watcher only re-tints through `setPageConfig`; the app forces
  `COLOR_MODE_LIGHT` in `onCreate`, so dark mode only arrives via
  `onConfigurationUpdate`.
- `common/`, `utils/` and `FontFileInfo` are unreferenced. Do not treat them
  as part of the pattern.

## Pitfalls

- **`HW-11-0026`** (B/low, confirmed): the auto-turn interval is not stopped in
  `aboutToDisappear`, so it survives page destruction and keeps calling
  `flipPage(true)` on a controller whose book has already been released. Fix:
  add `this.stopAutoScroll();` as the first line of `aboutToDisappear`.
- **`HW-11-0027`** (D/medium, confirmed): a page-turning reader declares
  `ohos.permission.LOCATION`, `ohos.permission.APPROXIMATELY_LOCATION` and
  `ohos.permission.LOCATION_IN_BACKGROUND` with zero location code anywhere in
  the sample - copy-pasted `module.json5` config that hands anyone scaffolding
  from this template an app-review rejection. Fix: remove the three
  `requestPermissions` entries.
- **Unregistered listeners are unregistered again.** `display.off('change', ...)`
  is called with a callback that is permanently `null`, and both
  `readerComponentController.off(...)` calls target events the sample never
  subscribed to. Harmless here, misleading as a template.
- **The flip-mode radios are not state-bound.** `.checked(true)` /
  `.checked(false)` are literals, so the radio selection resets while
  `readerSetting.flipMode` does not.

## References

- `huawei_industry_tree/11_news_reading/docs/25_automatic_page_turn.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/automatic_page_turn-0000002415970301
- `documentation/harmonyos-guides/04_application-services/reader-read-page.md` - the prescribed `init` / `getDefaultHandler` / `registerBookParser` / `startPlay` order and the `aboutToDisappear` teardown
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/reader-read-page
- `documentation/harmonyos-references/06_application-services/reader-read-core.md` - `ReaderComponentController`, `flipPage`, `setPageConfig`, `releaseBook`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/reader-read-core
- `documentation/harmonyos-references/06_application-services/reader-api-readpagecomponent.md` - `ReadPageComponent` and its `readerCallback`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/reader-api-readpagecomponent
- `documentation/harmonyos-references/05_common-capabilities/js-apis-timer.md` - `setInterval` / `clearInterval` and background suspension
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-timer
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - `bindSheet`, `onWillDismiss`, `DismissSheetAction`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-slider.md` - `Slider` options and `onChange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-slider
- `NEWS-07` - the page-flip *mode* selector this sheet reuses
- `NEWS-22` - the bookshelf sample whose import pipeline and permissions this one inherited
