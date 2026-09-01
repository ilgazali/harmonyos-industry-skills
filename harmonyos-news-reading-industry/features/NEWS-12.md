---
id: NEWS-12
title: Volume-key page turning - a global inputConsumer shortcut wired to Reader Kit flipPage
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/12_volume_key_turn_page.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/volume_key_turn_page-0000002293620017
sample: huawei_industry_tree/11_news_reading/downloads/VolumeKeyPageTurn.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.InputKit", "@kit.PerformanceAnalysisKit", "@kit.ReaderKit"]
apis: [bookParser, common, display, fs, hilog, inputConsumer, readerCore, window, "inputConsumer.on('keyPressed')", "inputConsumer.off('keyPressed')", KeyPressedConfig, KeyCode, ReadPageComponent, "readerCore.ReaderComponentController", flipPage, releaseBook, "bookParser.getDefaultHandler", CustomContentDialog, "@Watch"]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-11-0013, HW-11-0031]
status: verified-with-fixes
---

## When to use

**Load this card when a hardware key has to drive an in-app action while the
app is in the foreground** - volume keys turning pages in a reader, a camera
key shooting, a headset key skipping a track. `inputConsumer.on('keyPressed')`
is the API, and its defining property is that it does not just observe the key,
it **consumes** it: the reference states that on a successful call "the
system's default response to the key event will be intercepted; that is,
system-level actions, such as volume adjustment, will no longer be triggered."

That single sentence is the whole reason this card exists. Registering the
shortcut takes six lines and works immediately; the difficulty is entirely in
knowing when to *stop*. A volume key that no longer changes the volume is not a
feature the user can debug, and the sample leaves exactly that state behind
when the reader closes (`HW-11-0013`).

The second half of the card is the Reader Kit boot sequence - copy the book out
of `rawfile` into the sandbox, parse it, initialise the controller, register the
parser, start at a spine index. That part is shared with every other reader in
this industry (`NEWS-24`, `NEWS-25`, `NEWS-14`) and is worth reading once here.

## Feature checklist

- A full-screen Reader Kit reading page rendering a bundled `read.txt` with a
  simulated page-flip animation.
- Tapping the page toggles a bottom settings bar (table of contents, brightness,
  settings icons).
- The settings icon opens a bottom sheet with a single 音量键翻页 (volume-key
  page turn) toggle, defaulting to off.
- Turning it on makes VOLUME_DOWN advance a page and VOLUME_UP go back.
- While it is on, the volume keys no longer change the system volume.
- Turning it off restores the system volume behaviour immediately.
- Leaving the reader restores the system volume behaviour (this is the part the
  sample gets wrong).

## Architecture

One `entry` module, five ArkUI files. There is no book model: the book is a
`rawfile` and the reader component owns everything about its contents.

```
entry/src/main/ets
├── common/Constants.ets            CONFIGURATION record of layout numbers (largely unused here)
├── entryability/EntryAbility.ets   full screen + avoid areas -> AppStorage
├── entrybackupability/
├── pages/ReaderPage.ets            @Entry: a Stack of EmulationView under BottomView; owns all state
├── utils/Logger.ets                hilog wrapper
└── views/BottomView.ets            settings bar + the CustomContentDialog holding the toggle
    views/EmulationView.ets         the reader component, the boot sequence, the key listeners
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that the toggle and the listener
registration are in different components, connected by `@Link` plus `@Watch`.
`ReaderPage` owns `enableVolumeKey: boolean`; `BottomView` gets it as a `@Link`
and a `Toggle` writes it; `EmulationView` gets it as
`@Link @Watch('enableVolumeKeyChange')` and reacts. The setting is a single
piece of state with one writer and one reactor, and neither component has to
know the other exists.

That shape is the right one because it makes the side effect follow the state
rather than the tap. Any future writer of `enableVolumeKey` - a preference
restored at launch, a remote config, a second entry point in a settings page -
gets the listener registration for free, because `@Watch` fires on the value,
not on the gesture. Wiring `inputConsumer.on` into the `Toggle`'s `onChange`
would have coupled it to one particular UI.

The same mechanism is what makes the missing cleanup so easy to miss:
`@Watch` fires on change, and page destruction is not a change.

## Implementation steps

1. **Copy the book from `rawfile` into the sandbox** before parsing - the
   parser needs a real path, and `rawfile` contents are not one. Open the
   sandbox file **once** and close it in `finally` (see the corrected snippet).
2. **Start the controller and the parser in parallel**:
   `Promise.all([bookParserHandler, initPromise])`, since neither depends on
   the other.
3. **Register the parser into the controller, then `startPlay(spineIndex,
   domPos)`** - the order matters, the engine cannot lay out without a parser.
4. **Hold the setting as one boolean on the page** and pass it down as `@Link`
   to both the writer and the reactor.
5. **React with `@Watch`, not with the toggle's `onChange`.**
6. **Use a distinct callback per key.** The reference is explicit: "Ensure that
   different callbacks are used for different key events. Otherwise, the
   subscription does not take effect."
7. **Map the directions the way the hardware reads**: VOLUME_DOWN is
   `flipPage(true)` (forward), VOLUME_UP is `flipPage(false)` (back) - down for
   next matches the thumb travelling down the spine of the device.
8. **Unregister on the way out, not only on toggle-off**: `inputConsumer.off`
   in `aboutToDisappear` alongside `releaseBook()` (`HW-11-0013`).

## Verified snippets

All snippets are from `VolumeKeyPageTurn.zip`. Corrected forms are marked.

**The toggle-driven registration — `entry/src/main/ets/views/EmulationView.ets`** (as shipped)

```typescript
import { inputConsumer, KeyCode } from '@kit.InputKit';

@Component
export struct EmulationView {
  @Link isMenuViewVisible: boolean;
  @Link @Watch('enableVolumeKeyChange') enableVolumeKey: boolean;
  private readerComponentController: readerCore.ReaderComponentController = new readerCore.ReaderComponentController();

  enableVolumeKeyChange() {
    if (this.enableVolumeKey) {
      try {
        let upOption: inputConsumer.KeyPressedConfig = {
          key: KeyCode.KEYCODE_VOLUME_UP,
          action: 1,            // 1 = key down; 0 would fire on release
          isRepeat: false       // long-press must not flip pages continuously
        };
        inputConsumer.on('keyPressed', upOption, () => {
          this.readerComponentController.flipPage(false);   // back
        });
        let downOption: inputConsumer.KeyPressedConfig = {
          key: KeyCode.KEYCODE_VOLUME_DOWN,
          action: 1,
          isRepeat: false
        };
        inputConsumer.on('keyPressed', downOption, () => {
          this.readerComponentController.flipPage(true);    // forward
        });
      } catch (err) {
        let e = err as BusinessError;
        Logger.error(`onKeyPressed Failed: code = ${e.code}, message = ${e.message}`);
      }
    } else {
      try {
        inputConsumer.off('keyPressed');                    // drops ALL registered callbacks
      } catch (err) {
        let e = err as BusinessError;
        Logger.error(`offKeyPressed Failed: code = ${e.code}, message = ${e.message}`);
      }
    }
  }
}
```

**Three fields of `KeyPressedConfig` carry the behaviour.** `action: 1`
subscribes to key-down, so the page turns while the finger is still on the
button - key-up would feel laggy and would fire after a long-press menu. And
`isRepeat: false` is the one that saves the feature: with repeat enabled,
holding the volume key would flip pages at the auto-repeat rate, which in a
paginated reader means the user loses their place instantly.

The two arrow functions are separate objects, which satisfies the reference's
requirement that "different callbacks are used for different key events" -
passing the same function object for both keys silently drops the second
subscription. That is why this cannot be refactored into a loop over
`[[VOLUME_UP, false], [VOLUME_DOWN, true]]` that shares one handler.

The off branch calls `inputConsumer.off('keyPressed')` with no callback, which
the reference defines as "listening will be disabled for all registered
callbacks". Convenient here, dangerous in an app that registers other
shortcuts elsewhere - keep the callback references and pass them to `off` if
anything else in the app uses `inputConsumer`.

**Releasing on exit — same file** (corrected, see `HW-11-0013`)

```typescript
aboutToDisappear(): void {
  // FIX: absent in the sample - the global shortcut outlives the reader
  if (this.enableVolumeKey) {
    try {
      inputConsumer.off('keyPressed');
    } catch (err) {
      let e = err as BusinessError;
      Logger.error(`offKeyPressed Failed: code = ${e.code}, message = ${e.message}`);
    }
  }
  // 退出需要释放阅读器实例 - the reader instance must be released on exit
  this.readerComponentController.releaseBook();
}
```

**This is a four-line fix for a defect the user experiences as broken
hardware.** Leave the reader with the toggle on and two things are true at
once: the volume keys are still intercepted app-wide, so they no longer change
the volume anywhere in the app; and each press calls `flipPage` on a controller
whose book was released one line later in this very method. The user's model is
"the volume buttons stopped working", with no path back except killing the app.

`aboutToDisappear` is the right hook precisely because `@Watch` cannot cover
it - `enableVolumeKey` does not change when the page is destroyed, so the
reactor never runs. The mirror of this fix is re-registration: if the setting
is persisted, `aboutToAppear` must call `enableVolumeKeyChange()` once so a
reader re-entered with the toggle already on rebuilds its subscriptions.

The same "listener registered in one place, released in a narrower place than
it was registered" shape recurs across this industry: `HW-11-0026`
(auto-scroll interval, `NEWS-25`), `HW-11-0025` (`keyboardHeightChange`,
`NEWS-23`), `HW-11-0016` (typewriter interval, `NEWS-15`), `HW-11-0010`
(TextReader listeners, `NEWS-09`).

**Booting the reader — same file** (as shipped)

```typescript
private async startPlay(filePath: string, domPos: string) {
  try {
    let context = this.getUIContext().getHostContext() as common.UIAbilityContext;
    // controller init: lets ReadPageComponent talk to the typesetting engine
    let initPromise = this.readerComponentController.init(context);
    let bookParserHandler = bookParser.getDefaultHandler(filePath);
    let spineList: bookParser.SpineItem[] = (await bookParserHandler).getSpineList();
    let spineIndex: number = spineList[0].index;
    let result: [bookParser.BookParserHandler, void] = await Promise.all([bookParserHandler, initPromise]);
    this.bookParserHandler = result[0];
    // default typesetting style
    this.readerComponentController.setPageConfig(this.readerSetting);
    // hand the parser to the engine
    this.readerComponentController.registerBookParser(this.bookParserHandler);
    // open the book at the given chapter and offset
    this.readerComponentController.startPlay(spineIndex || 0, domPos);
  } catch (err) {
    hilog.error(0x0000, 'testTag', 'startPlay: err: ' + JSON.stringify(err));
  }
}
```

**The ordering is the API contract, not a style choice.** `init` and
`getDefaultHandler` are independent, so they are awaited together; but
`setPageConfig` must precede `registerBookParser`, and both must precede
`startPlay`, because the engine lays out the first page during `startPlay` and
needs the style and the parser already in place.

`readerSetting` is worth reading for its own sake: `scaledDensity` and
`viewPortWidth`/`viewPortHeight` come from
`display.getDefaultDisplaySync()`, with `scaledDensity` guarded to `1` when the
platform reports `0`. Those three are what the typesetting engine paginates
against - get them wrong and the page-count is wrong. Note they are read once
at construction, so a fold, rotation or window resize does not re-paginate.

`flipMode: '0'` selects the simulated page-curl animation; the other flip modes
(and the auto-scroll variant) are what `NEWS-24` and `NEWS-25` demonstrate.

**Copying the book to the sandbox — same file** (corrected — unfiled fd leak, see Pitfalls)

```typescript
copyRawfileToSanBox(context: common.UIAbilityContext, bookName: string): string {
  let bookSandBoxPath = context.filesDir + '/' + bookName;
  let file: files.File | undefined = undefined;
  try {
    let data = context.resourceManager.getRawFileContentSync(bookName);
    let buffer = data.buffer;
    // FIX: the sample opens the same path twice (once before the try, once here)
    //      and closes only the first handle in `finally`.
    file = files.openSync(bookSandBoxPath, files.OpenMode.CREATE | files.OpenMode.READ_WRITE);
    files.writeSync(file.fd, buffer);
  } catch (err) {
    let e = err as BusinessError;
    Logger.error(`copy book rawfile to sanbox failed: code = ${e.code}, message = ${e.message}`);
  } finally {
    if (file !== undefined) {          // FIX: the sample dereferences `file` unconditionally
      files.close(file.fd);
    }
  }
  return bookSandBoxPath;
}
```

**Two defects in eight lines, both of the kind that only show under load.** The
shipped code declares `let file = files.openSync(...)` *before* the `try`, then
declares a second, shadowing `let file = files.openSync(...)` on the same path
inside it. The inner handle is the one written to and the one that is never
closed; the outer handle is closed in `finally` having done nothing. One
descriptor leaks per call. Opening once, inside the `try`, and closing the same
handle in `finally` fixes both.

The `finally` block also runs when `openSync` itself throws, at which point the
sample's `file` is unassigned - the same unguarded-`finally` shape filed as
`HW-11-0023` against `NEWS-22`. Initialise to `undefined` and guard.

The copy is also unconditional: every entry into the reader re-copies the whole
book over the sandbox file. For a bundled sample text that is invisible; for a
real book it is worth an `accessSync` check first.

## Permissions & config

**None.** The sample declares no `requestPermissions`. `inputConsumer`'s
`keyPressed` subscription is not permission-gated for a foreground app - it is
scoped by focus instead: the reference notes the callback fires only when "the
current application is in the foreground focus window".

`deviceTypes` is `phone` only, and the ability declares
`"supportWindowMode": ["fullscreen"]`. Both are deliberate: `inputConsumer.off`
has device restrictions (below API 23 it only works on phones and tablets, and
returns error 801 elsewhere), and a floating or split-screen reader would not
reliably hold the focus the subscription depends on.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `inputConsumer.on('keyPressed')` / `off('keyPressed')` require API 16 or
  later.
- Device coverage differs between the two calls. `on` is available on phone,
  PC/2in1, tablet, TV and wearable; `off` works only on phones and tablets
  before API 23 (PC/2in1 and TV from API 23), and returns error code 801
  elsewhere. Registering where you cannot unregister is not a state to be in.
- The subscription is bound to foreground focus. Backgrounding the app suspends
  it; it does not cancel it.
- The reader's viewport is captured once from `display.getDefaultDisplaySync()`
  at component construction, so there is no re-pagination on rotation, fold or
  window resize.
- The settings sheet holds exactly one switch; brightness and table-of-contents
  icons in `BottomView` are decorative and have no handlers.
- The bundled book is `entry/src/main/resources/rawfile/read.txt`; there is no
  book picker. For import from the file picker see `NEWS-22`.

## Pitfalls

- **`HW-11-0013` — the global volume-key listener is not unregistered when the
  reader closes** (B/medium, confirmed). `inputConsumer.on` is called when the
  user enables the mode and `inputConsumer.off` only in the toggle-off branch;
  `aboutToDisappear` calls `releaseBook()` and nothing else. Leaving the reader
  with the toggle on keeps intercepting volume keys app-wide - so the volume no
  longer adjusts anywhere - and every press calls `flipPage` on a released
  controller. Fix: call `inputConsumer.off('keyPressed')` in
  `aboutToDisappear`, and re-register from `aboutToAppear` if the setting is
  persisted and on.
- **Not filed: `copyRawfileToSanBox` opens the sandbox file twice and closes
  the wrong handle.** The outer `let file` is opened before the `try`, an inner
  shadowing `let file` is opened inside it and written to, and `finally` closes
  only the outer one - one leaked descriptor per entry into the reader.
- **Not filed: the `finally` block dereferences `file` unconditionally,** so a
  throwing `openSync` turns a handled failure into a second exception. Same
  shape as `HW-11-0023` (`NEWS-22`).
- **Not filed: `avoidAreaChange` is registered in `EntryAbility` and never
  released** in `onWindowStageDestroy`. Recurring boilerplate defect; compare
  `HW-09-0021`.
- **Not filed: `CONFIGURATION` in `common/Constants.ets` is a 20-entry record
  of page-flip constants of which this sample uses none** - it is copied
  wholesale from the `PageFlipPage` sample (`NEWS-07`), whose own doc
  mis-quotes one of these names (`HW-11-0009`).

## References

- `documentation/harmonyos-references/03_system/js-apis-inputconsumer.md` -
  `on('keyPressed')`, `off('keyPressed')`, `KeyPressedConfig`, the
  interception note and the device-behaviour differences
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-inputconsumer
- `documentation/harmonyos-references/03_system/errorcode-inputconsumer.md` -
  subscription error codes
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/errorcode-inputconsumer
- `documentation/harmonyos-references/06_application-services/reader-api.md` -
  Reader Kit surface
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/reader-api
- `documentation/harmonyos-references/06_application-services/reader-read-core.md` -
  `ReaderComponentController`, `flipPage`, `startPlay`, `releaseBook`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/reader-read-core
- `documentation/harmonyos-references/06_application-services/reader-api-readpagecomponent.md` -
  `ReadPageComponent` and its `readerCallback`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/reader-api-readpagecomponent
- `NEWS-25` - the auto page-turn reader, and the same unreleased-timer defect
  (`HW-11-0026`)
- `NEWS-22` - book import through the file picker, where the sandbox copy is
  done for real
