---
id: COMMON-36
title: Foldable web layout switching - serve a desktop or mobile page by swapping the user agent on fold state
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/36_web_display_mode_switch.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_display_mode_switch-0000002366467641
sample: huawei_industry_tree/19_common_technical_solutions/downloads/WebDisplayModeSwitch.zip
kits: ["@kit.ArkWeb", "@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: ["webview.WebviewController", "WebviewController.setCustomUserAgent", "WebviewController.loadUrl", "Web.onControllerAttached", "Web.mixedMode", "Web.onlineImageAccess", "Web.zoomAccess", "Web.domStorageAccess", "Web.fileAccess", "Web.geolocationAccess", MixedMode, "display.isFoldable", "display.getFoldStatus", "display.on('foldStatusChange')", "display.off('foldStatusChange')", "display.FoldStatus", "display.getDefaultDisplaySync", "WindowStage.getMainWindow", "Window.getWindowProperties", "Window.on('windowSizeChange')", "Window.off('windowSizeChange')", "AppStorage.setOrCreate", "@StorageLink", "UIContext.px2vp", "component .expandSafeArea"]
permissions: ["ohos.permission.INTERNET"]
min_api: 20
modules: [entry]
findings: [HW-19-0108, HW-19-0109, HW-19-0110, HW-19-0182, HW-19-0183]
status: verified-with-fixes
---

## When to use

Load this card when the application shows a **remote web page inside a `Web`
component on a foldable**, and the page should render its desktop layout when the
device is unfolded (or the window is wide) and its mobile layout when folded or
in split screen.

The lever is the **user agent**: the server decides which layout to send, so the
application does not restyle anything - it re-declares what kind of client it is
and reloads.

## Feature checklist

The application must:

- Publish the live window width and height into `AppStorage`.
- Publish half the physical display width as the wide/narrow threshold.
- Track whether the device is foldable and what its current fold status is.
- Subscribe to window size changes and to fold-status changes - each **once** -
  and release both (HW-19-0108).
- Choose one of the layouts from (window width vs threshold) x (foldable) x
  (fold status).
- Set the matching user agent **before** loading the page, then load it
  (HW-19-0109).
- Lock the web view down: no zoom, no file access, no geolocation, and
  `MixedMode.None`.

## Architecture

Single-module project (`entry` HAP), two source files:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | `getDisplayWidth()`, the window and fold subscriptions, and everything published into `AppStorage` |
| `pages/WebPage.ets` | the three-branch layout selector and the `Web` components |

**Four values cross the boundary**, all through `AppStorage`:

| Key | Written by | Read by |
| --- | --- | --- |
| `winWidth`, `winHeight` | ability, on start and on every resize | page, via `@StorageLink`; also used to size the outer `Column` |
| `foldableScreenMiddleWidth` | ability, as `getDisplayWidth() / 2` | page, as the wide/narrow threshold |
| fold status | page itself, via `display.getFoldStatus()` | page |

Note that `isFoldable` and `foldStatus` are read in the page directly from
`display`, not through `AppStorage` - so the two halves of the decision come from
different places.

**The decision has three branches:**

| Condition | Layout | User agent |
| --- | --- | --- |
| `winWidth > threshold && isFoldable && foldStatus !== FOLD_STATUS_FOLDED` | wide | Windows / Chrome desktop |
| `winWidth <= threshold && isFoldable && foldStatus === FOLD_STATUS_FOLDED` | mobile | Android / HuaweiBrowser mobile |
| anything else (non-foldable, or a mismatch) | mobile | same mobile UA |

The second and third branches are identical apart from `expandSafeArea`, so in
practice the switch is binary: desktop when unfolded and wide, mobile otherwise.

**Why the first branch loads differently.** It passes `src: ''` and calls
`setCustomUserAgent` then `loadUrl` inside `onControllerAttached` - the ordering
the reference requires. The other two pass the URL as `src` and set the UA
afterwards, which is the ordering the reference warns about (HW-19-0109).

## Implementation steps

1. **Publish the window geometry** in `onWindowStageCreate`:
   `getMainWindow().then(w => { AppStorage.setOrCreate('winWidth',
   w.getWindowProperties().windowRect.width); ... })`.
2. **Compute the threshold** from the physical display:
   `display.getDefaultDisplaySync().width / 2`, logging any failure rather than
   returning a silent 0 (HW-19-0110).
3. **Subscribe once to each event**, at the same level - not one inside the other
   - and keep the callbacks so they can be released (HW-19-0108).
4. **Release both** in `onWindowStageDestroy`.
5. **Read the fold state in the page**: `display.isFoldable()` once, and
   `display.getFoldStatus()` into `@State` refreshed from a `foldStatusChange`
   subscription registered in `aboutToAppear` and released in
   `aboutToDisappear`.
6. **Branch on the three conditions** and render the appropriate `Web`.
7. **In every branch**: empty `src`, then in `onControllerAttached`
   `setCustomUserAgent(ua)` followed by `loadUrl(url)` (HW-19-0109).
8. **Harden the web view** in every branch: `zoomAccess(false)`,
   `fileAccess(false)`, `geolocationAccess(false)`, `mixedMode(MixedMode.None)`.

## Verified snippets

All snippets below come from the sample project, not from the document.

### Publishing the geometry (as shipped - see HW-19-0108 and HW-19-0110)

`WebDisplayModeSwitch.zip#WebDisplayModeSwitch/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
getDisplayWidth(): number {
  let displayClass: display.Display | null = null;
  let foldableScreenWidth: number = 0;
  try {
    displayClass = display.getDefaultDisplaySync();
    foldableScreenWidth = displayClass.width;
    return foldableScreenWidth;
  } catch (exception) {
    foldableScreenWidth = 0;      // FIX (HW-19-0110): log the exception
    return foldableScreenWidth;
  }
}

onWindowStageCreate(windowStage: window.WindowStage): void {
  windowStage.getMainWindow().then((windowClass) => {
    // 获取窗口尺寸，存入AppStorage
    AppStorage.setOrCreate('winWidth', windowClass.getWindowProperties().windowRect.width);
    AppStorage.setOrCreate('winHeight', windowClass.getWindowProperties().windowRect.height);
    // 监听窗口尺寸变化
    windowClass.on('windowSizeChange', (windowSize) => {
      AppStorage.setOrCreate('winWidth', windowSize.width);
      AppStorage.setOrCreate('winHeight', windowSize.height);
      AppStorage.setOrCreate('foldableScreenMiddleWidth', this.getDisplayWidth() / 2);

      display.on('foldStatusChange', (data: display.FoldStatus) => {   // FIX (HW-19-0108): hoist out
        AppStorage.setOrCreate('foldableScreenMiddleWidth', this.getDisplayWidth() / 2);
        AppStorage.setOrCreate('winWidth', windowClass.getWindowProperties().windowRect.width);
      })

    });
  });

  windowStage.loadContent('pages/WebPage', (err) => {
    if (err.code) {
      hilog.error(DOMAIN, 'testTag', 'Failed to load the content. Cause: %{public}s', JSON.stringify(err));
      return;
    }
    hilog.info(DOMAIN, 'testTag', 'Succeeded in loading the content.');
  });
}
```

### The page state and the fold subscription

`WebDisplayModeSwitch.zip#WebDisplayModeSwitch/entry/src/main/ets/pages/WebPage.ets`

```ts
@Entry
@Component
struct Index {
  private webController: web_webview.WebviewController = new web_webview.WebviewController()
  @StorageLink('winWidth') winWidth: number = 0;
  @StorageLink('winHeight') winHeight: number = 0;
  @StorageLink('foldableScreenMiddleWidth') foldableScreenMiddleWidth: number = 0;
  private uiContext: UIContext = this.getUIContext();
  private isFoldable: boolean = display.isFoldable();
  @State foldStatus: display.FoldStatus = display.getFoldStatus();

  onPageShow(): void {                          // FIX (HW-19-0108): aboutToAppear + off in aboutToDisappear
    display.on('foldStatusChange', (data: display.FoldStatus) => {
      this.foldStatus = display.getFoldStatus();
    })
  }
}
```

### The wide branch - the correct load ordering

`WebDisplayModeSwitch.zip#WebDisplayModeSwitch/entry/src/main/ets/pages/WebPage.ets`

```ts
if (this.winWidth > this.foldableScreenMiddleWidth && this.isFoldable &&
  this.foldStatus !== display.FoldStatus.FOLD_STATUS_FOLDED) {
  Column() {
    Web({ src: '', controller: this.webController })
      .backgroundColor(Color.Transparent)
      .width('100%')
      .height('100%')
      .zoomAccess(false)
      .domStorageAccess(true)
      .onlineImageAccess(true)
      .mixedMode(MixedMode.None)
      .geolocationAccess(false)
      .fileAccess(false)
      .onControllerAttached(() => {
        // Windows系统用UA
        this.webController.setCustomUserAgent('Mozilla/5.0 (Windows NT 10.0; Win64; x64) ' +
          'AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36')
        this.webController.loadUrl($r('app.string.uri_example'))
      })
  }
}
```

### The mobile branch - as shipped (see HW-19-0109)

`WebDisplayModeSwitch.zip#WebDisplayModeSwitch/entry/src/main/ets/pages/WebPage.ets`

```ts
} else {
  Column() {
    Web({ src: $r('app.string.uri_example'), controller: this.webController })
      .backgroundColor(Color.Transparent)
      .width('100%')
      .height('100%')
      .zoomAccess(false)
      .domStorageAccess(true)
      .mixedMode(MixedMode.None)
      .fileAccess(false)
      .geolocationAccess(false)
      .onControllerAttached(() => {
        // 移动端通用UA
        this.webController.setCustomUserAgent('Mozilla/5.0 (Linux; Android 9; VRD-AL10; HMSCore 6.3.0.331) ' +
          'AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.88 HuaweiBrowser/12.0.4.1 Mobile Safari/537.36')
      })
      .expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.TOP])
  }
}
```

### Sizing the container from the published window geometry

`WebDisplayModeSwitch.zip#WebDisplayModeSwitch/entry/src/main/ets/pages/WebPage.ets`

```ts
.width('100%')
.size({
  width: this.uiContext.px2vp(this.winWidth),
  height: this.uiContext.px2vp(this.winHeight)
})
```

The published values are px, converted at the point of use - the right way round.

## Permissions & config

`ohos.permission.INTERNET` is required and is the only permission declared - the
page is remote.

`WebDisplayModeSwitch.zip#WebDisplayModeSwitch/entry/src/main/module.json5`:

```json5
"requestPermissions":[
  {
    "name" : "ohos.permission.INTERNET"
  }
]
```

The target URL lives in a string resource (`app.string.uri_example`), which is
why it can be handed to `loadUrl` as a `Resource` rather than a string.

Neither `display.isFoldable()`, `display.getFoldStatus()` nor
`display.on('foldStatusChange')` needs a permission.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`setCustomUserAgent` must precede the load.** "You are advised to call the
  setCustomUserAgent method to set a user agent before using loadUrl to load a
  specific page", because calling it during the first load "causes the component
  to reload and retain the initial history entry", leaving `accessBackward`
  false.
- **The layout comes from the server.** This technique only works for sites that
  serve different markup per user agent; a responsive site that keys off viewport
  width will ignore the change entirely.
- **`display.isFoldable()` is a device property**, read once; `getFoldStatus()`
  is live and must be re-read on every `foldStatusChange`.
- **The threshold is half the *display* width, not half the window.** In split
  screen the window is narrower than half the display, which is what makes the
  narrow branch reachable without folding.
- **`mixedMode(MixedMode.None)`** blocks http subresources on an https page -
  keep it; a desktop UA often makes the server return more third-party content.
- **Devices.** Meaningful on foldables; `isFoldable()` is false everywhere else,
  so a non-foldable always takes the third branch.

## Pitfalls

- **`display.on('foldStatusChange')` is registered inside the
  `windowSizeChange` handler, which is incorrect.** Every resize adds another
  listener and none are ever removed, so after n resizes one fold event runs the
  handler n times. The page adds a second leak by registering in `onPageShow`
  with no `off`. (HW-19-0108)
- **Two of the three branches set the user agent after the page has started
  loading, which is incorrect.** The reference warns this replaces the initial
  history entry and breaks `accessBackward`; the sample's own first branch shows
  the correct empty-src + `setCustomUserAgent` + `loadUrl` sequence.
  (HW-19-0109)
- **`getDisplayWidth`'s catch returns 0 silently, which is incorrect.** A zero
  threshold makes `winWidth > threshold` true for every window, so a folded
  device gets the desktop layout - and nothing is logged to explain it.
  (HW-19-0110)
- **`getMainWindow()` has no rejection handler.** If it rejects, no geometry is
  published, `winWidth` stays 0, and the outer `Column` is sized to zero.
- **The second and third branches are duplicates.** They differ only by
  `expandSafeArea`, so the middle condition earns nothing; collapsing them would
  make the real decision - wide versus narrow - obvious.
- **The fold state is read from two places.** `isFoldable` / `foldStatus` come
  straight from `display` in the page, while the width threshold arrives through
  `AppStorage` from the ability; a fold that updates one before the other renders
  one frame with a mismatched pair.
- **Spoofing a desktop user agent is a server-visible choice.** Some sites treat
  an unexpected UA as a bot; verify against the actual target site before
  shipping.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-webview-webviewcontroller.md` -
  `setCustomUserAgent` and the history/`accessBackward` caveat, `loadUrl`,
  `onControllerAttached`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-webview-webviewcontroller
- `documentation/harmonyos-references/02_application-framework/js-apis-display.md` -
  `isFoldable`, `getFoldStatus`, `FoldStatus`, `on`/`off('foldStatusChange')`,
  `getDefaultDisplaySync`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-display
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `getWindowProperties`, `on`/`off('windowSizeChange')`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-attributes.md` -
  `mixedMode`, `zoomAccess`, `onlineImageAccess`, `fileAccess`,
  `geolocationAccess`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-basic-components-web-attributes
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-state-management -
  `AppStorage` and `@StorageLink`.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_display_mode_switch-0000002366467641
