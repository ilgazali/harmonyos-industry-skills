---
id: NEWS-07
title: Three page-turn modes in one reader - Swiper, List and Reader Kit behind one switch
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/07_page_flip_page.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/page_flip_page-0000002271210553
sample: huawei_industry_tree/11_news_reading/downloads/PageFlipPage.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit", "@kit.ReaderKit"]
apis: [bookParser, common, display, emitter, fs, hilog, readerCore, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-11-0009, HW-11-0031]
status: verified-with-fixes
---

## When to use

Load this card when a reading surface has to offer **more than one way to
advance the page** - left-right paging, vertical scrolling, and a simulated
page curl - and the user picks between them from a settings sheet. It is the
standard reader affordance in news apps with long-form articles, in ebook
readers, and in anything paginating serialised content.

The transferable idea is that the three modes are **three different
components, not three modes of one component**. Left-right is a `Swiper`,
up-down is a `List`, and the curl is Reader Kit's `ReadPageComponent` - which
brings its own typesetting engine and cannot be talked into behaving like the
other two. The page holds one string naming the current mode and swaps the
component; the current page number is the only state that crosses.

Read the Reader Kit half separately from the other two. `Swiper` and `List`
render ArkUI text you already have. `ReadPageComponent` renders a *book*: you
must first get a file into the app sandbox, hand it to a parser, register the
parser with a controller, and release the controller on exit. If your content
is a string in memory, the first two modes are the whole feature.

## Feature checklist

- A reading page with a fixed header (章节 title and a subtitle line) and a
  bottom bar that appears on a mid-screen tap.
- The bottom bar's 设置 (settings) icon opens a sheet with three capsule
  buttons: 左右翻页, 上下翻页, 仿真翻页.
- Picking one swaps the reading surface immediately and closes the sheet; the
  active capsule is drawn filled and slightly shorter.
- Left-right mode: tapping the right third advances, the left third goes back,
  the middle third toggles the menus; swipe works too; no indicator dots.
- Up-down mode: continuous vertical scroll with the scrollbar hidden.
- Both track which page is showing and restore it when the mode changes.
- Simulated mode: a book copied out of `rawfile` into the sandbox, typeset by
  the Reader Kit engine, with a page-curl animation.
- The reader instance is released when the component disappears.

## Architecture

One `entry` module. One page holds the mode string and every piece of shared
state; five view components consume it through `@Link`.

```
entry/src/main/ets
├── common/Constants.ets            CONFIGURATION (numbers) + STRINGCONFIGURATION (names, resource keys)
├── datasource/BasicDataSource.ets  IDataSource over string[]; add/push/insert/delete + listeners
├── entryability/EntryAbility.ets   full-screen, avoid areas -> AppStorage, avoidAreaChange subscription
├── entrybackupability/EntryBackupAbility.ets
├── pages/Index.ets                 @Entry: the mode string, page number, font/colour, rawfile copy
├── utils/Logger.ets
└── views/
    ├── BottomView.ets              the menu bar and the mode-picker CustomContentDialog
    ├── EmulationView.ets           Reader Kit: controller + parser + ReadPageComponent
    ├── LeftRightFlipView.ets       Swiper + LazyForEach + tap-zone paging
    ├── TopView.ets                 fixed header, status-bar avoidance
    └── UpDownFlipView.ets          List + LazyForEach
```

The documented tree matches the zip exactly.

**The design decision worth copying** is the mode switch itself. `Index.ets`
holds one string and an `if/else if/else` chain in `build()`:

```typescript
if (this.buttonClickedName === STRINGCONFIGURATION.LEFT_RIGHT_FLIP_PAGE_NAME) {
  LeftRightPlipPage({ ..., currentPageNum: this.currentPageNum, ... })
} else if (this.buttonClickedName === STRINGCONFIGURATION.UP_DOWN_FLIP_PAGE_NAME) {
  UpDownFlipPage({ ..., currentPageNum: this.currentPageNum })
} else {
  EmulationView({ isMenuViewVisible: this.isMenuViewVisible, currentPageNum: this.currentPageNum })
}
```

Because ArkUI destroys the branch that stops matching, switching modes runs
the outgoing component's `aboutToDisappear` - which is what releases the
Reader Kit book without any explicit teardown call. The page number is passed
as `@Link` into all three, so whichever component is alive writes it and the
next one reads it as its initial index. **One piece of shared state, three
independent renderers**: that is the whole coordination mechanism, and it is
the right amount for this problem.

The decision *not* to copy is the string identity: the modes are compared by
their Chinese display names (`'左右翻页'`, `'上下翻页'`, `'仿真翻页'`) held in
`STRINGCONFIGURATION`. Those strings are simultaneously the button labels, so
localising the UI silently breaks the switch. Use an enum for identity and a
resource for the label.

## Implementation steps

1. **Define the modes as identities, not labels**, and keep the page number in
   the parent so it survives the swap.
2. **For left-right, use `Swiper` + `LazyForEach`** over an `IDataSource`, with
   `cachedCount` set so neighbouring pages are typeset before the user reaches
   them, and `indicator(false)` because a reader has its own progress row.
3. **Bind `Swiper.index` to `currentPageNum - PAGE_FLIP_PAGE_COUNT`** and write
   it back in `onChange`. The offset exists because pages are numbered from 1
   and indices from 0.
4. **For up-down, use `List({ initialIndex: ... })` with the same offset** and
   `onScrollIndex` instead of `onChange`. Use the sample's constant name
   `CONFIGURATION.PAGE_FLIP_PAGE_COUNT`; the document's snippet writes
   `CONFIGURATION.PAGEFLIPPAGECOUNT`, which exists nowhere in the project
   (`HW-11-0009`).
5. **Divide the reading surface into three tap zones** by comparing `event.x`
   against thirds of the measured screen width, and halve that width above the
   600 vp breakpoint so a two-column tablet layout still hits the right zone.
6. **For simulated paging, copy the book out of `rawfile` first.**
   `resourceManager.getRawFileContent` gives you bytes; write them into
   `context.filesDir`. Reader Kit parses a sandbox path, not a resource.
7. **Initialise controller and parser in parallel, then order the three calls:**
   `setPageConfig` -> `registerBookParser` -> `startPlay(spineIndex, domPos)`.
8. **Release the book in `aboutToDisappear`.** `releaseBook()` frees the engine
   instance; without it, switching modes back and forth leaks readers.

## Verified snippets

All snippets are from `PageFlipPage.zip`. Corrected forms are marked.

**Left-right paging — `entry/src/main/ets/views/LeftRightFlipView.ets`** (as shipped)

```typescript
private swiperController: SwiperController = new SwiperController();
private data: BasicDataSource = new BasicDataSource([]);
private screenW: number = this.uiContext.px2vp(display.getDefaultDisplaySync().width);

aboutToAppear(): void {
  for (let i = CONFIGURATION.PAGE_FLIP_PAGE_START; i <= CONFIGURATION.PAGE_FLIP_PAGE_END; i++) {
    this.data.pushItem(STRINGCONFIGURATION.PAGE_FLIP_RESOURCE + i.toString());
  }
  if (this.screenW > CONFIGURATION.WINDOW_WIDTH) {
    this.screenW = this.screenW / 2;
  }
}

build() {
  Column() {
    Swiper(this.swiperController) {
      LazyForEach(this.data, (item: string) => {
        Text($r(item))
          .width($r('app.string.pageflip_full_size'))
          .height($r('app.string.pageflip_full_size'))
          .fontSize(this.contentFontsize)
          .fontColor(this.contentFontColor)
          .textAlign(TextAlign.Start)
          .lineHeight(33);
      }, (item: string) => item);
    }
    .index(this.currentPageNum - CONFIGURATION.PAGE_FLIP_PAGE_COUNT)
    .indicator(false)
    .cachedCount(CONFIGURATION.PAGE_FLIP_CACHE_COUNT)
    .loop(false)
    .curve(Curve.Linear)
    .duration(CONFIGURATION.PAGE_FLIP_TOAST_DURATION)
    .onChange((index: number) => {
      this.currentPageNum = index + CONFIGURATION.PAGE_FLIP_PAGE_COUNT;
    })
  }
  .expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.TOP, SafeAreaEdge.BOTTOM])
  .onClick((event?: ClickEvent) => {
    if (!event) { return; }
    if (this.isMenuViewVisible) { this.isMenuViewVisible = false; }
    if (event.x > this.screenW / CONFIGURATION.PAGE_FLIP_THREE * CONFIGURATION.PAGE_FLIP_TWO) {
      if (this.currentPageNum !== this.data.totalCount()) { this.swiperController.showNext(); }
    } else if (event.x > this.screenW / CONFIGURATION.PAGE_FLIP_THREE) {
      this.isMenuTopVisible = !this.isMenuTopVisible;
      if (this.isMenuViewVisible) { this.isMenuTopVisible = false; }
      else { this.isMenuViewVisible = true; this.isMenuTopVisible = true; }
    } else {
      if (this.currentPageNum !== CONFIGURATION.PAGE_FLIP_PAGE_START) { this.swiperController.showPrevious(); }
    }
  });
}
```

**Four options make this a reader rather than a carousel.** `indicator(false)`
removes the dots - a book with hundreds of pages cannot show them.
`loop(false)` stops page N wrapping to page 1, which for a reader is a bug,
not a feature. `cachedCount(3)` keeps neighbours typeset so the turn is
instant, and pairs with `LazyForEach` so distant pages are never built.
`curve(Curve.Linear)` with a 300 ms `duration` gives a flat slide; the
easing curves that suit a photo carousel read as sluggish under repeated taps.

**The tap zones are computed, not hardcoded.** `screenW` comes from
`display.getDefaultDisplaySync().width` converted with `px2vp`, then halved
above the 600 vp breakpoint - the assumption being that a wide window shows
two pages side by side, so the interactive width per page is half the window.
The three-way split (`> 2/3` next, `> 1/3` menu, else previous) is the
convention readers use, and putting the menu in the middle band is what lets
both edges stay pure paging targets.

**Vertical paging — `entry/src/main/ets/views/UpDownFlipView.ets`** (as shipped; note the constant name, `HW-11-0009`)

```typescript
List({ initialIndex: this.currentPageNum - CONFIGURATION.PAGE_FLIP_PAGE_COUNT }) {
  LazyForEach(this.data, (item: string) => {
    ListItem() {
      Text($r(item))
        .fontSize($r('app.integer.flippage_text_fontsize'))
        .width($r('app.string.pageflip_full_size'))
        .lineHeight(33)
        .padding({ left: 24, right: 20 });
    };
  }, (item: string) => item);
}
.scrollBar(BarState.Off)
.cachedCount(CONFIGURATION.PAGE_FLIP_CACHE_COUNT)
.onScrollIndex((firstIndex: number) => {
  this.currentPageNum = firstIndex + CONFIGURATION.PAGE_FLIP_PAGE_COUNT;
});
```

**The identifier is `PAGE_FLIP_PAGE_COUNT`, underscored.** The document's
snippet writes `CONFIGURATION.PAGEFLIPPAGECOUNT`, which appears nowhere in the
project; copying the documented line against the sample's `Constants.ets`
fails to compile (`HW-11-0009`). The constant's value is `1`, and its job is to
convert between 1-based page numbers and 0-based indices - the same offset the
`Swiper` applies, which is what lets the page number survive a mode switch.

`initialIndex` versus `Swiper`'s `index` is the substantive difference between
the two modes. `List.initialIndex` is read once at construction; `Swiper.index`
tracks. That works here only because the mode switch destroys and rebuilds the
component, so "once at construction" happens exactly when the page number
needs to be restored.

**Getting the book into the sandbox — `entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
myRawfileCopy(context: common.UIAbilityContext) {
  context.resourceManager.getRawFileContent('read.txt', (err: BusinessError, data: Uint8Array) => {
    if (err != null) {
      Logger.error('open file failed: ' + err.message);
    } else {
      let buffer = data.buffer;
      let file = fs.openSync(context.filesDir + '/read.txt', fs.OpenMode.CREATE | fs.OpenMode.READ_WRITE);
      try {
        fs.writeSync(file.fd, buffer);           // 拷贝文件到沙箱
      } catch (err) {
        Logger.info('myRawfileCopy error');
      } finally {
        fs.close(file.fd);
      }
    }
  });
}

aboutToAppear(): void {
  let context = this.uiContext.getHostContext();
  this.myRawfileCopy(context as common.UIAbilityContext);
}
```

**Reader Kit parses a filesystem path, never a resource.** `rawfile` contents
live inside the HAP and have no sandbox path, so the bytes have to be copied
out once into `context.filesDir` before `bookParser.getDefaultHandler` can be
pointed at them. The copy runs from the *page's* `aboutToAppear`, not the
reader component's, so the file is on disk before the user can ever reach the
simulated mode - deliberate, and the reason `EmulationView` can build its path
synchronously.

Two things to change when you adopt this: the copy is unconditional, so it
rewrites the file on every page entry (add an `fs.accessSync` check), and it is
callback-based while everything downstream is promise-based, so nothing can
await it. Make it `async` and await it in the reader's own `aboutToAppear` if
the book might be large.

**The Reader Kit reader — `entry/src/main/ets/views/EmulationView.ets`** (as shipped)

```typescript
import { bookParser, readerCore, ReadPageComponent } from '@kit.ReaderKit';

private readerComponentController: readerCore.ReaderComponentController =
  new readerCore.ReaderComponentController();
// 默认设置项，用于初始化阅读器页面的默认属性。
private readerSetting: readerCore.ReaderSetting = {
  fontName: '', fontPath: '', fontSize: 15, fontColor: '#000000', fontWeight: 4,
  lineHeight: 1.7, nightMode: true, themeColor: '#EAE2CF', themeBgImg: '',
  flipMode: '0',
  scaledDensity: display.getDefaultDisplaySync().scaledDensity > 0
    ? display.getDefaultDisplaySync().scaledDensity : 1,
  viewPortWidth: 1216, viewPortHeight: 2490,
};
private bookParserHandler: bookParser.BookParserHandler | null = null;

aboutToAppear(): void {
  let filePath: string = ((this.uiContext.getHostContext()) as common.UIAbilityContext).filesDir + '/read.txt';
  this.startPlay(filePath, '');
}

private async startPlay(filePath: string, domPos: string) {
  try {
    let context = this.getUIContext().getHostContext() as common.UIAbilityContext;
    let initPromise = this.readerComponentController.init(context);
    let bookParserHandler = bookParser.getDefaultHandler(filePath);
    let spineList: bookParser.SpineItem[] = (await bookParserHandler).getSpineList();
    let spineIndex: number = spineList[0].index;
    let result: [bookParser.BookParserHandler, void] = await Promise.all([bookParserHandler, initPromise]);
    this.bookParserHandler = result[0];
    this.readerComponentController.setPageConfig(this.readerSetting);
    this.readerComponentController.registerBookParser(this.bookParserHandler);
    this.readerComponentController.startPlay(spineIndex || 0, domPos);
  } catch (err) {
    hilog.error(0x0000, 'testTag', 'startPlay: err: ' + JSON.stringify(err));
  }
}

aboutToDisappear(): void {
  this.readerComponentController.releaseBook();   // 退出需要释放阅读器实例
}

build() {
  Stack() {
    ReadPageComponent({
      controller: this.readerComponentController,
      readerCallback: (err: BusinessError, data: readerCore.ReaderComponentController) => {
        this.readerComponentController = data;
      }
    });
  }
  .onClick((event?: ClickEvent) => { if (event) { this.isMenuViewVisible = !this.isMenuViewVisible; } });
}
```

**The call order is a contract, not a preference.** `init(context)` must have
resolved and the parser must be registered before `startPlay`, because
`startPlay` asks the engine to typeset a spine item it can only reach through
the registered parser. The sample overlaps the two independent awaits with
`Promise.all([bookParserHandler, initPromise])` - the controller init and the
book parse do not depend on each other, so running them concurrently saves the
slower of the two. `setPageConfig` comes before `registerBookParser` so the
first typeset already uses the intended font and viewport.

**`readerCallback` reassigns the controller.** The component hands back the
controller instance it actually bound to, and the sample stores it over its own
field. That matters for `releaseBook()` on the way out: releasing the
pre-construction instance would leave the bound one alive.

`scaledDensity` is read from the display with a `> 0` fallback to `1`, which is
the one piece of this settings block that is environment-derived rather than
hardcoded. `viewPortWidth: 1216` / `viewPortHeight: 2490` are px for one
reference device and are the first values to replace with measured ones - see
Constraints.

## Permissions & config

**None.** The sample declares no `requestPermissions`; the book comes from its
own `rawfile` and is copied into its own sandbox.

`module.json5` declares `deviceTypes: ["phone", "tablet", "2in1"]` and an
`EntryBackupAbility` extension with `$profile:backup_config`.

`EntryAbility` sets `COLOR_MODE_LIGHT`, calls `setWindowLayoutFullScreen(true)`
with both handlers attached, seeds `topRectHeight` and `bottomRectHeight` into
`AppStorage`, and subscribes to `avoidAreaChange` to keep them current.
`TopView` consumes `topRectHeight` through `@StorageProp` and converts with
`px2vp` at the point of use - the safe form of that pattern.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **`ReadPageComponent` is available from API 5.0.4(16)** and is documented for
  Phone, PC/2in1 and Tablet only. There is no wearable story for the simulated
  mode; guard by device type if the app targets watches.
- **The viewport is hardcoded.** `viewPortWidth: 1216` and
  `viewPortHeight: 2490` are pixel dimensions for one phone. On a tablet or a
  resized 2in1 window the engine typesets to the wrong page box. Derive them
  from the component's measured size before shipping.
- **`nightMode: true` sits next to a light `themeColor: '#EAE2CF'`**, and the
  ability forces `COLOR_MODE_LIGHT`. The combination is inconsistent; decide
  which one owns the theme before copying the settings block.
- **The "book" is a 1.6KB `read.txt`.** It exercises the API surface, not the
  engine - pagination, chapter navigation and progress restore are not
  meaningfully demonstrated at that size. `domPos` is always `''`, so the
  reader always opens at the start of the first spine item; a real reader
  stores the last `domPos` and passes it back.
- **The emitter wiring is dead code.** `Index.ets` defines `registerEmitter`
  (subscribing to event id 2 to leave full-screen on a back press) but never
  calls it, and `aboutToDisappear` calls `deleteEmitter()`, which unsubscribes
  event id **1** - a third id that is never used. Nothing breaks, because
  nothing was subscribed; delete all three or wire them up consistently.
- The three modes do not share font settings. `contentFontsize` and
  `contentColor` reach `LeftRightPlipPage` only; `UpDownFlipPage` reads its
  size from `$r('app.integer.flippage_text_fontsize')`, and `EmulationView`
  from `readerSetting.fontSize`. Changing the size in one mode does not carry
  to the others.
- `BottomView`'s comment records that its middle region has no components, so
  taps there fall through to the reading surface underneath - which is why the
  menu toggle also lives in the reading component's own `onClick`.

## Pitfalls

- **`HW-11-0009` (E/low, confirmed) — the document's up-down snippet
  references `CONFIGURATION.PAGEFLIPPAGECOUNT`, an identifier that does not
  exist.** The zip's `UpDownFlipView.ets:42` uses
  `CONFIGURATION.PAGE_FLIP_PAGE_COUNT`, and no un-underscored spelling appears
  anywhere in the project, so the documented line does not compile against the
  sample's constants file. Fix: rename the constant in the snippet to
  `CONFIGURATION.PAGE_FLIP_PAGE_COUNT`.

Beyond the recorded finding, three shapes in this sample will bite anyone who
copies it as-is, all covered under Constraints: the hardcoded Reader Kit
viewport, the unconditional rawfile copy on every page entry, and the mode
identity being the localised button label.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-swiper.md` - `SwiperController`, `index`, `cachedCount`, `loop`, `indicator`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-swiper
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `initialIndex`, `onScrollIndex`, `cachedCount`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` - `IDataSource` and the listener protocol `BasicDataSource` implements
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `documentation/harmonyos-references/06_application-services/reader-api-readpagecomponent.md` - `ReadPageComponent`, its `controller`/`readerCallback` parameters, and the reference init/`releaseBook` sequence
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/reader-api-readpagecomponent
- `documentation/harmonyos-guides/07_application-services/reader-kit-guide.md` - book parsing, typesetting and interaction
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/reader-kit-guide
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `openSync`, `writeSync`, `close` for the rawfile copy
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `NEWS-08` - the same avoid-area boilerplate, applied to a list rather than a reader
- `huawei_industry_tree/11_news_reading/docs/07_page_flip_page.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/page_flip_page-0000002271210553
