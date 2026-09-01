---
id: AUTO-03
title: Persistent bottom notification bar via a window-stage sub-window
industry: 01_auto
doc: huawei_industry_tree/01_auto/docs/03_global_components.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/global_components-0000002313328742
sample: huawei_industry_tree/01_auto/downloads/GlobalComponents.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: [WindowStage.createSubWindow, Window.setUIContent, Window.moveWindowTo, Window.resize, Window.showWindow, Window.destroyWindow, Window.setWindowBackgroundColor, Window.getWindowAvoidArea, Window.on avoidAreaChange, display.getDefaultDisplaySync, onVisibleAreaChange]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-01-0013, HW-01-0014, HW-01-0015, HW-01-0016, HW-01-0017, HW-01-0018, HW-01-0019, HW-01-0050]
status: verified-with-fixes
---

## When to use

Load this card when a strip of UI must **stay pinned above every page of the
app** and survive navigation - a persistent notification bar, an ongoing-call
banner, a now-playing bar, a "your car is charging" strip. The mechanism is a
**window-stage sub-window**, not an in-page overlay, which is why it floats over
whatever page is showing without every page having to render it.

Do not reach for this when an in-page overlay would do. A sub-window is a real
window with its own lifecycle, its own UI content, and its own destruction path;
`bindSheet`, a `Stack` overlay or `bindContentCover` are cheaper and safer for
anything scoped to one page.

## Feature checklist

- A bar fixed near the bottom of the screen, above the page content.
- Visible regardless of which page or tab is showing.
- Its own content page, loaded independently of the host page.
- A close button that destroys the window.
- Position that respects the device's navigation-bar inset.

## Architecture

Single `entry` module.

```
entry/src/main/ets
├── common/Constants.ets          sizes, colours, the sub-window geometry
├── entryability/EntryAbility.ets measures avoid areas into AppStorage
├── pages/MainPage.ets            tab host; opens the sub-window
├── pages/FloatPage.ets           the sub-window's own UI content
├── pages/BuyingCarPage.ets       tab content
├── pages/MinePage.ets            tab content
└── utils/GlobalComponents.ets    the sub-window wrapper (singleton)
```

Three pieces:

1. **`EntryAbility`** puts the main window into full-screen layout, reads the
   system and navigation-indicator avoid areas, publishes them to `AppStorage`
   as `topRectHeight` / `bottomRectHeight`, and subscribes to `avoidAreaChange`
   to keep them current.
2. **`GlobalComponents.ets`** exports a singleton (`export default new
   Loading()`) holding one `window.Window` handle, with `OpenSubWindows` and
   `CloseSubWindows`.
3. **`FloatPage`** is the sub-window's content, loaded by path
   (`'pages/FloatPage'`), and it calls back into the singleton to close itself.

The sub-window is a **sibling of the main window under the same window stage**,
not a child of any component. That is what makes it survive page navigation, and
also what makes its lifecycle entirely manual.

## Implementation steps

1. **Get the `UIAbilityContext`** in the page:
   `this.getUIContext().getHostContext() as common.UIAbilityContext`, and reach
   `context.windowStage` from it.
2. **Call `createSubWindow(name, callback)` inside a `try`** - it throws
   synchronously on parameter errors (`HW-01-0016`).
3. **In the callback, check `err.code` first, then null-check the window**,
   then store the handle.
4. **Configure in order**: `setUIContent(path, cb)` → set the background colour
   in that callback → `moveWindowTo` → `resize` → `showWindow`.
5. **Derive the position from the measured avoid area**, not from a magic pixel
   offset (`HW-01-0014`).
6. **Guard creation on the existing handle** so repeated calls are idempotent
   (`HW-01-0013`).
7. **On close, await `destroyWindow()`, catch its rejection, and clear the
   handle.**
8. **Release `avoidAreaChange` in `onWindowStageDestroy`** (`HW-01-0015`).

## Verified snippets

All snippets are from `GlobalComponents.zip`. Corrected forms are marked.

**The sub-window wrapper — `entry/src/main/ets/utils/GlobalComponents.ets`** (corrected, see `HW-01-0013` and `HW-01-0016`)

```typescript
import { common } from '@kit.AbilityKit';
import { display, window } from '@kit.ArkUI';
import { hilog } from '@kit.PerformanceAnalysisKit';
import { Constants } from '../common/Constants';

export class Loading {
  win: window.Window | undefined = undefined;

  OpenSubWindows(context: common.UIAbilityContext) {
    if (this.win) {
      return;                       // FIX: shipped code has no guard and overwrites the handle
    }
    try {                           // FIX: shipped try sits inside the callback, so a
                                    // synchronous throw from createSubWindow escapes
      context.windowStage.createSubWindow('FloatPage', (err, windowClass) => {
        if (err.code > 0) {
          hilog.error(0x0000, 'testTag', '%{public}s', `failed to create subWindow: ${err.message}`);
          return;
        }
        if (!windowClass) {         // FIX: shipped code dereferences without checking
          hilog.error(0x0000, 'testTag', '%{public}s', 'subWindow is null');
          return;
        }
        this.win = windowClass;
        try {
          windowClass.setUIContent('pages/FloatPage', () => {
            windowClass.setWindowBackgroundColor(Constants.SUBWINDOW_BACKGROUND_COLOR);
          });
          const bottom = AppStorage.get<number>('bottomRectHeight') ?? 0;
          const y = display.getDefaultDisplaySync().height - bottom - Constants.SUB_WINDOW_HEIGHT;
          windowClass.moveWindowTo(Constants.ZERO, y);   // FIX: shipped code uses height - 370
          windowClass.resize(display.getDefaultDisplaySync().width, Constants.SUB_WINDOW_HEIGHT);
          windowClass.showWindow();
        } catch (err) {
          hilog.error(0x0000, 'testTag', '%{public}s', `failed to configure subWindow: ${err}`);
        }
      });
    } catch (exception) {
      hilog.error(0x0000, 'testTag', '%{public}s', `createSubWindow threw: ${JSON.stringify(exception)}`);
    }
  }

  async CloseSubWindows() {
    try {
      await this.win?.destroyWindow();   // FIX: shipped code neither awaits nor catches
    } catch (err) {
      hilog.error(0x0000, 'testTag', '%{public}s', `failed to destroy subWindow: ${err}`);
    } finally {
      this.win = undefined;              // FIX: shipped code leaves a stale handle
    }
  }
}

export default new Loading();
```

The singleton export is the right shape: one handle for one window, reachable
from both the page that opens it and the content page that closes it.

**Measuring the avoid areas — `entry/src/main/ets/entryability/EntryAbility.ets`** (as shipped, plus the missing cleanup)

```typescript
onWindowStageCreate(windowStage: window.WindowStage): void {
  let windowClass: window.Window = windowStage.getMainWindowSync();
  windowClass.setWindowLayoutFullScreen(true).then(() => {
    hilog.info(0x0000, 'testTag', 'Succeeded in setting full-screen layout.');
  });

  let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR;
  let avoidArea = windowClass.getWindowAvoidArea(type);
  AppStorage.setOrCreate('bottomRectHeight', avoidArea.bottomRect.height);

  type = window.AvoidAreaType.TYPE_SYSTEM;
  avoidArea = windowClass.getWindowAvoidArea(type);
  AppStorage.setOrCreate('topRectHeight', avoidArea.topRect.height);

  windowClass.on('avoidAreaChange', (data) => {
    if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
      AppStorage.setOrCreate('topRectHeight', data.area.topRect.height);
    } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
      AppStorage.setOrCreate('bottomRectHeight', data.area.bottomRect.height);
    }
  });
  this.windowClass = windowClass;     // FIX: keep it so it can be released
}

onWindowStageDestroy(): void {
  this.windowClass?.off('avoidAreaChange');   // FIX: shipped hook only logs
}
```

This is the piece worth lifting wholesale: full-screen layout plus both avoid
areas in `AppStorage`, kept live by `avoidAreaChange`. Consume it anywhere with
`@StorageProp('bottomRectHeight') bottomRectHeight: number = 0` and
`this.getUIContext().px2vp(this.bottomRectHeight)`.

Note the units: avoid-area rects are **px**, so convert with `px2vp` before
using them as layout values, and do **not** convert before passing them to
`moveWindowTo` / `resize`, which take px.

**Opening it from a page — `entry/src/main/ets/pages/MainPage.ets`** (corrected, see `HW-01-0013` and `HW-01-0018`)

```typescript
@State context: common.UIAbilityContext = this.getUIContext().getHostContext() as common.UIAbilityContext;

aboutToAppear(): void {
  Loading.OpenSubWindows(this.context);
}

// FIX: the shipped code also opens from onVisibleAreaChange with no guard,
// and its not-visible branch logs "sub window is visible".
.onVisibleAreaChange([0.0, 1.0], (isVisible: boolean, currentRatio: number) => {
  if (isVisible && currentRatio >= 1.0) {
    Loading.OpenSubWindows(this.context);
  }
  if (!isVisible && currentRatio <= 0.0) {
    Loading.CloseSubWindows();      // symmetric: this is what the branch was for
  }
});
```

Pick one strategy: either open once in `aboutToAppear` and close on page
destruction, or open and close symmetrically from `onVisibleAreaChange`. The
sample does half of both.

**The sub-window's content page — `entry/src/main/ets/pages/FloatPage.ets`** (as shipped)

```typescript
@Entry
@Component
struct FloatPage {
  build() {
    Navigation(this.pageInfos) {
      Row({ space: Constants.SPACE_EIGHT }) {
        Image($r('app.media.ic_bottom')).width(...).height(...);
        Text($r('app.string.news')).fontSize(...).fontColor('#000000');
        Blank();
        Image($r('app.media.close'))
          .onClick(() => {
            this.isClosed = true;
            Loading.CloseSubWindows();
          });
      }
      .width(Constants.FULL_PERCENT)
      .height(Constants.BOTTOM_HEIGHT)
      .backgroundColor('#EBF2FE');
    }
    .hideTitleBar(true)
    .hideToolBar(true);
  }
}
```

`FloatPage` is `@Entry` because it is loaded by path through `setUIContent`, not
rendered as a component - this is one of the few places `@Entry` is genuinely
required. It must be registered in `main_pages.json` like any routed page.

## Permissions & config

**None.** `entry/src/main/module.json5` declares no `requestPermissions` - a
sub-window under your own window stage needs no permission. (Floating over
*other* apps is a different capability and does require
`ohos.permission.SYSTEM_FLOAT_WINDOW`, which is a restricted permission needing
approval. This practice does not use it.)

`entry/src/main/ets/common/Constants.ets`:

```typescript
static readonly SUB_WINDOW_HEIGHT: number = 100;   // px
static readonly ZERO: number = 0;
// Do not copy the shipped SUB_WINDOW_Y - see HW-01-0014.
```

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `moveWindowTo` and `resize` take **px**, while layout values are vp - mixing
  them is the easiest way to get this wrong.
- The sub-window's content is a routed page, so it must be listed in
  `main_pages.json`.
- The window is entirely manual: nothing destroys it when the opening page goes
  away. Every creation path needs a matching destruction path.
- `createSubWindow` names must be unique within the window stage; reusing a live
  name reports an error through the callback.

## Pitfalls

- **`HW-01-0013` — creation is not idempotent and closing leaves a stale
  handle.** `OpenSubWindows` overwrites `this.win` unconditionally and is called
  from both `aboutToAppear` and `onVisibleAreaChange`; `CloseSubWindows` neither
  awaits nor catches `destroyWindow()` and never clears the handle. A window
  whose handle gets overwritten can never be destroyed.
- **`HW-01-0014` — `SUB_WINDOW_Y = display.height - 370`.** A magic offset,
  computed once at module load, on an app that already measures the real
  navigation-bar inset and publishes it to `AppStorage`. Wrong height on any
  other device geometry, and frozen across rotation and fold.
- **`HW-01-0015` — `avoidAreaChange` is never released.** `onWindowStageDestroy`
  exists but only logs, so listeners accumulate across ability restarts.
- **`HW-01-0016` — `createSubWindow` can throw synchronously.** The reference
  example wraps the *call* in `try`; the sample's `try` is inside the callback,
  and it never null-checks the window the callback hands back.
- **`HW-01-0017` — the document's snippet does not compile.** It shows
  `OpenSubWindows()` reading `this.context`, a member the class does not have,
  and it deletes both `hilog.error` calls, leaving an empty `catch` for readers
  to copy.
- **`HW-01-0018` — the not-visible branch logs "sub window is visible".** It is
  also where the missing `CloseSubWindows()` belongs.
- **`HW-01-0019` — four of five tabs render `BuyingCarPage()`,** including the
  ones the project tree labels home and service, and there is no home page in
  the sample at all.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-windowstage.md` - `createSubWindow`, and the try/catch plus null-check pattern the sample omits
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-windowstage
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `moveWindowTo`, `resize`, `showWindow`, `destroyWindow`, `getWindowAvoidArea`, `on('avoidAreaChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - `ohos.permission.SYSTEM_FLOAT_WINDOW`, needed only for floating over *other* apps, not for this practice
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `AUTO-01` - the industry architecture this component drops into
