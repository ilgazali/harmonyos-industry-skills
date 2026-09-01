---
id: NEWS-09
title: AI read-aloud - drive the SpeechKit TextReader panel from one static util and a floating play button
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/09_ai_recitation.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/ai_recitation-0000002290573329
sample: huawei_industry_tree/11_news_reading/downloads/AIRecitation.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit", "@kit.SpeechKit"]
apis: [display, hilog, window, "TextReader.init", "TextReader.start", "TextReader.on", "TextReader.showPanel", "TextReader.showMinibar", "TextReader.hideMinibar", "TextReader.loadMore", "TextReader.playNext", "TextReader.playPrev", ReadStateCode, "windowStage.createSubWindow", "window.findWindow", "window.shiftAppWindowFocus"]
permissions: [ohos.permission.INTERNET, ohos.permission.GET_NETWORK_INFO, ohos.permission.KEEP_BACKGROUND_RUNNING]
min_api: 20
modules: [entry (entry)]
findings: [HW-11-0010, HW-11-0028, HW-11-0031]
status: verified-with-fixes
---

## When to use

**Load this card when the app has to read text aloud and you do not want to
build a player.** SpeechKit's `TextReader` ships the whole playback surface -
a full panel with chapter list, speed and voice pickers, plus a minibar that
docks over the reading page. The app's job is reduced to three things: hand
over an array of `ReadInfo` chapters, start playback at one of their ids, and
keep its own page in step with the panel through the `stateChange` callback.

The pattern generalises well past novels. Any long-form reader - news
articles, mail digests, a policy document, a recipe - has the same shape: a
list of addressable text blocks, one of which is "current", and a player that
can move the cursor by itself when a chapter finishes. The load-bearing idea
is that **the panel is authoritative**. The user can change chapter from
inside the system UI, so the app has to treat `setArticle` and `stateChange`
as inputs, not confirmations of its own commands.

**Read `HW-11-0010` before adopting it.** The sample registers a `requestMore`
listener whose body does nothing at all, and never releases the reader. Both
are one-line problems whose consequences (silent dead-end when the panel asks
for more chapters, a speech service left initialised on exit) are exactly what
a production reader must not have.

## Feature checklist

- A novel reading page with a header, the chapter body, and a bottom bar with
  previous/next chapter arrows that slides in when the page is tapped.
- Tapping the page while nothing is playing raises a round floating button
  over the bottom-right of the screen.
- Tapping that floating button registers the listeners, starts playback of the
  current chapter and opens the full `TextReader` panel.
- Chapter changes made inside the panel move the reading page to the same
  chapter.
- Chapter changes made with the page's own arrows move the panel, but only
  while it is playing.
- While playback is running, tapping the page shows the system minibar instead
  of the app's floating button.
- The floating window minimises itself as soon as the panel takes over, and
  hands focus back to the main window.

## Architecture

One `entry` module, six ArkUI files. There is no data layer: both chapters are
literal strings, once inside `TextReaderUtil` (for the reader) and once as
string resources (for the page).

```
entry/src/main/ets
├── entryability/EntryAbility.ets   full-screen layout; publishes windowStage + mainWindowId to AppStorage
├── entrybackupability/
├── pages/NovelReadingPage.ets      @Entry; the reading page, the subwindow lifecycle, the tap router
├── utils/TextReaderUtil.ets        the ReadInfo array, init, the three listeners, the state callback
├── views/BottomView.ets            previous/next chapter arrows, slid off-screen by a negative margin
├── views/TopView.ets               book title + author line
└── window/PlayWindow.ets           @Entry of the floating subwindow: one round button that starts playback
```

The documented tree does **not** match the zip: the doc lists `view`, the zip
directory is `views`. This is the same doc defect filed as `HW-11-0020`
against `NEWS-18`; it is unreported for this document. Everything else lines
up.

**The design decision worth copying** is that `TextReaderUtil` is a static
class, not an instance. Three unrelated UI surfaces need the same two facts -
which chapter is selected and whether it is playing - and they live in
different windows: `NovelReadingPage` and `BottomView` are in the main window,
`PlayWindow` is in a subwindow with its own UI instance. `@State`, `@Link` and
even `AppStorage` are per-instance or per-ability plumbing that would need
threading through both windows; a static class is process-wide and needs
none. `TextReaderUtil.selectedReadID` and `TextReaderUtil.readState` are read
directly from all three.

**And one worth avoiding.** Because the static fields are plain statics rather
than observed state, the page cannot react to them. The sample bridges the gap
with `setInterval(..., 100)` in `aboutToAppear`, polling `selectedReadID` ten
times a second forever - it is never cleared, and it re-renders the page on a
timer instead of on a change. The right shape is an `AppStorage` key written
from `onStateChanged` and read with `@StorageLink` in the page, which removes
both the timer and the leak.

## Implementation steps

1. **Build the `ReadInfo[]` once** with a stable `id` per chapter. The ids are
   the panel's addressing scheme: `start`, `setArticle` and every `ReadState`
   are all keyed by them, so use the article's real identity, not the array
   index as this sample does.
2. **Initialise before you can be asked to play.** `TextReader.init` is async;
   gate the play affordance on a flag it sets (`isTextReaderInit` here) so a
   tap that arrives during init cannot call `start` on an uninitialised
   reader.
3. **Register the listeners immediately before `start`,** in one place, so the
   panel never emits into a void.
4. **Give `requestMore` a real loader** - the panel raises it when it runs out
   of queued chapters, and the answer is `TextReader.loadMore(next, isEnd)`,
   not a discarded function reference (`HW-11-0010`).
5. **Route `stateChange` through one handler** that compares the event id with
   the app's current id: same id means a state transition to record, different
   id means the panel moved the user to another chapter and the app must
   follow.
6. **Release on the way out.** Call `TextReader.off(...)` for each event and
   `TextReader.release()` in `aboutToDisappear`/`onDestroy` (`HW-11-0010`).
7. **Show exactly one play affordance.** While `readState` is
   `ReadStateCode.PLAYING`, use the system minibar and keep the app's floating
   window minimised; otherwise show the floating window and hide the minibar.
8. **Create the floating button as a subwindow, not a dialog,** and hand focus
   straight back to the main window in its `onPageShow` so the reading page
   keeps receiving taps.
9. **Do not declare `KEEP_BACKGROUND_RUNNING` unless you start a continuous
   task** (`HW-11-0028`).

## Verified snippets

All snippets are from `AIRecitation.zip`. Corrected forms are marked.

**Initialising the panel — `entry/src/main/ets/utils/TextReaderUtil.ets`** (as shipped)

```typescript
import { ReadStateCode, TextReader } from '@kit.SpeechKit';

export class TextReaderUtil {
  static readState: ReadStateCode;
  static readInfoList: TextReader.ReadInfo[] = [{
    id: '0',
    title: { text: '第一章：穿越', isClickable: true },      // chapter 1: "Time travel"
    author: { text: 'AI', isClickable: true },
    date: { text: '2024/01/01', isClickable: false },
    bodyInfo: '在21世纪初的一个春日午后……'
  }, {
    id: '1',
    title: { text: '第二章：科学', isClickable: true },      // chapter 2: "Science"
    author: { text: 'AI', isClickable: true },
    date: { text: '2024/01/01', isClickable: false },
    bodyInfo: '增加新鲜蔬菜和水果的摄入……'
  }];
  static selectedReadID: string = TextReaderUtil.readInfoList[0].id;

  static async init(context: Context) {
    const READER_PARAM: TextReader.ReaderParam = {
      isVoiceBrandVisible: true,
      businessBrandInfo: {
        panelName: '书籍名',        // the caption shown at the top of the panel
      },
      displayTab: 2,               // 2 = detail tab only; no chapter-list tab
      minibarParams: {
        defaultAlignment: 1,
        bottom: 177
      }
    };
    await TextReader.init(context, READER_PARAM);
  }
}
```

**Three fields of `ReaderParam` carry the whole panel design.** `displayTab: 2`
collapses the panel to the detail view, which is the right call when the app
already owns the chapter list - two competing chapter pickers is the usual
mistake here. `minibarParams.bottom: 177` reserves vertical room so the minibar
docks above the app's own bottom bar instead of on top of it; it is a vp offset
from the bottom edge, so it must be re-derived if the bottom bar changes
height. `businessBrandInfo.panelName` is what makes the system panel read as
part of this app rather than a generic OS player.

Note that `ReadInfo.title`, `author` and `date` are objects with an
`isClickable` flag, not strings: the panel renders them as tappable chips when
the flag is set, and the app receives the tap through the panel's own
callbacks.

**The listeners — same file** (corrected, see `HW-11-0010`)

```typescript
  // Loads the article list into the panel
  static setEventListener() {
    TextReader.on('eventReadList', (event: Array<TextReader.ListEventState>) => {
      hilog.info(0x0001, TAG, `ReadList event: ${JSON.stringify(event)}`);
      TextReader.loadMore([], true);
    });
  }

  static setActionListener() {
    TextReader.on('setArticle', (id: string) => {
      TextReaderUtil.selectedReadID = id;              // the panel moved the user
    });
    TextReader.on('stateChange', (state: TextReader.ReadState) => {
      hilog.info(0x1, TAG, `ReadState: %{public}s`, JSON.stringify(state));
      TextReaderUtil.onStateChanged(state);
    });
    // FIX: shipped code is `TextReader.on('requestMore', () => TextReaderUtil.onStateChanged);`
    //      - the arrow body evaluates a method reference and throws it away.
    TextReader.on('requestMore', () => {
      TextReader.loadMore([], true);                   // no further chapters in this sample
    });
  }

  // FIX: absent in the sample - nothing ever releases the reader
  static release() {
    TextReader.off('eventReadList');
    TextReader.off('setArticle');
    TextReader.off('stateChange');
    TextReader.off('requestMore');
    TextReader.release();
  }

  static onStateChanged(state: TextReader.ReadState) {
    if (TextReaderUtil.selectedReadID === state.id) {
      TextReaderUtil.readState = state.state;          // same chapter: record the transition
    } else {
      TextReaderUtil.selectedReadID = state.id;        // different chapter: follow the panel
    }
  }
}
```

**`onStateChanged` is the synchronisation contract, and its asymmetry is
deliberate.** When the event's id matches what the app thinks is current, the
event is a state change on the known chapter and only `readState` moves. When
it does not match, the panel has advanced by itself - chapter finished,
auto-next - and the app adopts the new id instead of the new state. The state
for the new chapter arrives in the following event, at which point the ids
agree and the first branch runs.

`() => TextReaderUtil.onStateChanged` is worth staring at, because it compiles,
type-checks and does nothing: the arrow evaluates the static method as a value
and discards it. Any handler that has to be *invoked* must be either the
reference itself (`TextReaderUtil.onStateChanged`) or a call
(`() => TextReaderUtil.onStateChanged(x)`). Here neither is right anyway -
`requestMore` means "the queue is empty, send more", and its answer is
`loadMore`.

**Routing between minibar and floating button — `entry/src/main/ets/pages/NovelReadingPage.ets`** (as shipped)

```typescript
  aboutToAppear(): void {
    TextReaderUtil.init(this.uiContext.getHostContext() as Context).then(() => {
      this.isTextReaderInit = true;
    }).catch((e: BusinessError) => {
      hilog.error(0x0000, 'testTag', `TextReader failed to initialize. Code: ${e.code}, message: ${e.message}`);
    });
    setInterval(() => {                       // polling bridge; never cleared
      this.bookmark = Number(TextReaderUtil.selectedReadID);
    }, 100);
  }

  // ... on the article List:
  .onClick(() => {
    this.isMenuViewVisible = !this.isMenuViewVisible;
    if (this.isTextReaderInit) {
      if (this.isMenuViewVisible) {
        if (TextReaderUtil.readState === ReadStateCode.PLAYING) {
          TextReader.showMinibar();           // playing: the system minibar is the control
        } else {
          TextReader.hideMinibar();
          this.isWinShow();                   // idle: our own floating play button
        }
      } else {
        TextReader.hideMinibar();
        window.findWindow('PlayWindow').minimize();
      }
    }
  })
```

**One tap toggles three surfaces at once** - the app's top/bottom chrome, the
system minibar and the floating button - and the branch that decides which of
the last two appears is `readState`. That is the whole reason the state has to
be mirrored out of the panel: without it the app would show a play button over
an already-playing article.

`isTextReaderInit` is the init gate from step 2. The sample keeps a second,
duplicated branch for the not-yet-initialised case that only manages the
floating window, which is the right instinct - the button may appear before
the engine is ready, it simply cannot start anything yet.

**The floating play button — `entry/src/main/ets/window/PlayWindow.ets`** (as shipped)

```typescript
@Entry
@Component
struct PlayWindow {
  @State windowName: string = 'PlayWindow';
  @State windowStage: window.WindowStage = AppStorage.get('windowStage') as window.WindowStage;

  onPageShow(): void {
    let subWindowID: number = window.findWindow(this.windowName).getWindowProperties().id;
    let mainWindowID: number = this.windowStage.getMainWindowSync().getWindowProperties().id;
    window.shiftAppWindowFocus(subWindowID, mainWindowID);   // give focus straight back
  }

  build() {
    Row() {
      Column() {
        Row() {
          Text($r('app.string.liston'))
            .fontSize(40);
        }
        .backgroundColor('#80FFFFFF')
        .height(69)
        .width(69)
        .borderRadius(35)
        .shadow({ radius: 5, color: '#15000000' });
      };
    }
    .onClick(() => {
      TextReaderUtil.setActionListener();
      TextReaderUtil.setEventListener();
      TextReader.start(TextReaderUtil.readInfoList, TextReaderUtil.selectedReadID);
      TextReader.showPanel();
      window.findWindow('PlayWindow').minimize();
    });
  }
}
```

**`shiftAppWindowFocus` in `onPageShow` is the non-obvious line.** A subwindow
takes focus when it is shown, and a focused 73vp button would swallow the key
and back events that belong to the reading page. Shifting focus back to the
main window immediately leaves the button tappable but not focused - the
standard shape for a floating control.

The click handler is also the sample's registration point: listeners are
attached, then `start`, then `showPanel`. Registering right before `start`
means the panel cannot emit `stateChange` into a void, but it also means the
listeners are re-registered on every press of the button. In a real app,
register once (next to `init`) and release once (step 6); `TextReader.on`
does not de-duplicate for you.

`window.findWindow('PlayWindow')` looks the window up by name from anywhere in
the process, which is why the main page can minimise a window it did not
build. The subwindow itself is created in `NovelReadingPage.isWinShow()` with
`createSubWindow` + `moveWindowTo(display width - 73vp, 591vp)` + `resize`;
those coordinates are absolute device pixels via `vp2px`, so they place the
button correctly only on a phone-sized display.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET",           "reason": "$string:EntryAbility_desc", "usedScene": { "abilities": [] } },
  { "name": "ohos.permission.GET_NETWORK_INFO",   "reason": "$string:EntryAbility_desc", "usedScene": { "abilities": [] } },
  { "name": "ohos.permission.KEEP_BACKGROUND_RUNNING", "reason": "$string:background_permission_reason" }
],
"abilities": [{ "name": "EntryAbility", "backgroundModes": ["audioPlayback"], ... }]
```

- `INTERNET` and `GET_NETWORK_INFO` are needed because the TextReader voices are
  synthesised server-side; both are `system_grant`, so no runtime request
  appears.
- `KEEP_BACKGROUND_RUNNING` plus `backgroundModes: ["audioPlayback"]` is the
  correct *declaration* for a reader that should keep talking with the screen
  off - but a declaration alone does nothing. The continuous task only exists
  once `backgroundTaskManager.startBackgroundRunning` is called, and no such
  call exists anywhere in this sample. The identical defect is filed against
  the sibling TextReader sample as `HW-11-0028`; this project is a second
  instance.
- `usedScene: { "abilities": [] }` with an empty array is boilerplate here: the
  field is only meaningful for `user_grant` permissions, and neither of these
  is one.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `deviceTypes` is `phone`, `tablet`,
  `2in1`.
- The panel is a system surface: its layout, voices and speed controls are not
  themable beyond `ReaderParam`. If the product needs a bespoke player UI,
  this is the wrong API - use Core Speech Kit's TTS engine directly, as
  `NEWS-01` does.
- Playback needs the network. There is no offline path and no error surface in
  the sample for a failed synthesis.
- The floating window is positioned with `vp2px` against
  `display.getDefaultDisplaySync().width`, so it is placed for the default
  display only; on a folded/unfolded transition or a resized 2in1 window it
  will not move.
- `BottomView` compares `TextReaderUtil.readState === 1` with a numeric literal
  where the page uses `ReadStateCode.PLAYING`. Both mean the same today; use
  the enum in both places.
- Chapter navigation is bounded to the two hardcoded chapters
  (`bookmark < 1`, `bookmark > 0`) in `BottomView`.

## Pitfalls

- **`HW-11-0010` — the `requestMore` listener is a no-op and the reader is
  never released** (B/low, confirmed). `TextReader.on('requestMore', () =>
  TextReaderUtil.onStateChanged)` evaluates a method reference and discards
  it, so when the panel asks for more chapters nothing happens; and no
  `TextReader.off()` or `TextReader.release()` exists anywhere, so the speech
  service and its three listeners survive the page. Fix: make `requestMore`
  call a real loader (`TextReader.loadMore`), and release the reader in
  `aboutToDisappear`.
- **`HW-11-0028` — `KEEP_BACKGROUND_RUNNING` and `backgroundModes` are
  declared but no continuous task is ever started** (D/low, confirmed).
  Reported against the sibling TextReader sample (`NEWS-26`); the same
  declaration-without-implementation appears in this project's
  `module.json5`. Fix: either call
  `backgroundTaskManager.startBackgroundRunning` when playback starts, or drop
  the permission and the background mode.
- **Not filed: the 100 ms `setInterval` in `aboutToAppear` is never cleared.**
  It exists only to poll a static field; replace it with `AppStorage` +
  `@StorageLink` and the timer disappears with the leak. Same class as
  `HW-11-0016` (`NEWS-15`) and `HW-11-0026` (`NEWS-25`), which recur across
  this industry.
- **Not filed: the documented project tree says `view`, the zip directory is
  `views`.** Identical to `HW-11-0019` and `HW-11-0020`, filed against other
  documents in this industry.

## References

- `documentation/harmonyos-references/07_ai/speech-textreader-api.md` -
  `TextReader.init`, `ReaderParam`, `ReadInfo`, the `on` events
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/speech-textreader-api
- `documentation/harmonyos-guides/08_ai/speech-textreader-guide.md` - the
  panel/minibar integration flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/speech-textreader-guide
- `documentation/harmonyos-guides/03_application-framework/continuous-task.md` -
  what `KEEP_BACKGROUND_RUNNING` actually requires
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/continuous-task
- `documentation/harmonyos-guides/03_application-framework/window-overview.md` -
  subwindows, `createSubWindow`, focus shifting
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/window-overview
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` -
  `INTERNET`, `GET_NETWORK_INFO`, `KEEP_BACKGROUND_RUNNING`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `NEWS-01` - the same industry's hand-rolled TTS, for when the system panel is
  not wanted
- `NEWS-26` - the sibling TextReader sample and the origin of `HW-11-0028`
