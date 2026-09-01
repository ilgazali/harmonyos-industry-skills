---
id: NEWS-16
title: H5 font size - drive a Web component's textZoomRatio from the app's own reading-size setting
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/16_h5_fontsize.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5_fontsize-0000002289377882
sample: huawei_industry_tree/11_news_reading/downloads/WebFontSize.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.ArkWeb", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [Web, textZoomRatio, "webview.WebviewController", loadUrl, onControllerAttached, accessBackward, backward, domStorageAccess, javaScriptAccess, fileAccess, imageAccess, forceDisplayScrollBar, "preferences.getPreferencesSync", putSync, getSync, flush, NavDestination, NavPathStack, "UIContext.getPromptAction", showToast, expandSafeArea, onTouch, "resourceManager.getStringByNameSync"]
permissions: [ohos.permission.INTERNET]
min_api: 20
modules: [entry (entry)]
findings: [HW-11-0018, HW-11-0031]
status: verified-with-fixes
---

## When to use

Load this card when a reading app has **its own font-size control and also
shows web content** - a news app whose article body is an H5 page, a help
centre, a licence or terms page, an embedded blog. Without this, the user
raises the reading size in the app, taps through to a web article, and the text
snaps back to the browser default.

The mechanism is one attribute: `Web(...).textZoomRatio(n)`, where `n` is a
percentage of the page's own font sizes. Bind it to state, step the state from
the app's A+/A- controls, and the H5 page re-lays-out to match the surrounding
native UI. There is no JavaScript injection, no CSS override, and no
cooperation needed from the page being displayed - which is what makes the
technique work on content you do not control.

It generalises to any Web-embedded reading surface, and it composes with the
rest of an accessibility setting: the same stored percentage can scale native
`Text` through `fontSize` while scaling the Web through `textZoomRatio`.
**The persistence half of the sample does not work as documented** - read
`HW-11-0018` before treating this as a finished setting.

## Feature checklist

- A web article fills the page, loaded from a URL held in a string resource.
- A bottom bar carries 增大字体 / 减小字体 (increase / decrease), plus collect
  and share buttons.
- Tapping increase steps the zoom 100 → 125 → 150 percent; tapping decrease
  steps back down.
- At either bound a toast says the maximum or minimum has been reached, and the
  value does not move.
- Every icon has a pressed variant swapped in on touch-down and back out on
  touch-up.
- The back arrow walks the Web history first and only leaves the page when
  there is nothing to go back to.
- The chosen size is written to `preferences` and restored on the next visit.

## Architecture

One `entry` module, two ArkTS files besides the ability boilerplate.

```
entry/src/main/ets
├── common/CommonConstants.ets     preference key, URL resource name, step size, FontSizePreference enum
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
└── pages/WebPage.ets              @Entry - the Web, the bottom bar, the preferences handle
```

The documented tree matches the zip.

**The design decision worth copying** is that the zoom percentage is an
`enum` of *legal values*, not a free number:

```typescript
export enum FontSizePreference {
  FONT_SIZE_NORMAL = 100,
  FONT_SIZE_MEDIUM = 125,
  FONT_SIZE_LARGE = 150
}
```

`@State fontSizePct: FontSizePreference` is then typed by that enum, so the
stored setting, the state and the attribute all speak the same three-valued
vocabulary. The bounds checks compare against the enum members rather than
against magic numbers, and the step (`FONT_SIZE_INTERVAL = 25`) is exactly the
gap between adjacent members - the arithmetic and the enumeration agree by
construction. That is what a settings value should look like: a closed set the
UI can render as three labelled options later, not a slider that has to be
clamped.

The cost is that the invariant is unenforced. Change the interval to 20 and
`fontSizePct` silently takes values that are not enum members, while the type
still claims otherwise. If the set of sizes may grow, index into an array of
sizes instead of adding to the value.

## Implementation steps

1. **Hold a `webview.WebviewController`** as a plain field, and pass it to the
   `Web` component.
2. **Construct the `Web` with `src: ''` and load in `onControllerAttached`.**
   The controller's methods throw until it is bound to a component, so the
   attach callback is the earliest legal point to call `loadUrl`.
3. **Keep the URL in a string resource** and read it with
   `resourceManager.getStringByNameSync` - it changes per build flavour, not
   per user.
4. **Bind `textZoomRatio(this.fontSizePct)`.** The percentage applies to the
   page's own sizes, so 100 means "as authored".
5. **Enable only the Web capabilities the content needs** -
   `javaScriptAccess`, `domStorageAccess`, `imageAccess`, `fileAccess` are all
   switched on here; `fileAccess` in particular opens the app's file system to
   the page and should be off for remote content.
6. **Step within bounds and toast at the edge,** comparing against the enum
   members rather than raw numbers.
7. **Write the value with `putSync` and then `flush()`** (`HW-11-0018`).
   `putSync` alone only mutates the in-memory instance; nothing reaches disk.
8. **Read the stored value back in `aboutToAppear`** with
   `getSync(key, FONT_SIZE_NORMAL)` and assign it to the state. The shipped
   sample instead *writes* `FONT_SIZE_NORMAL` there, which would discard the
   user's choice even if step 7 were fixed.
9. **Intercept back through the controller:** `accessBackward()` then
   `backward()`, returning `true` when the Web consumed the gesture.
10. **Declare `ohos.permission.INTERNET`** - a `Web` loading a remote URL is
    dead without it.

## Verified snippets

All snippets are from `WebFontSize.zip`. Corrected forms are marked.

**The Web component — `entry/src/main/ets/pages/WebPage.ets`** (as shipped)

```typescript
url: string = (this.getUIContext()
  .getHostContext() as common.UIAbilityContext).resourceManager
  .getStringByNameSync(CommonConstants.URL_RESOURCE_NAME);

controller: webview.WebviewController = new webview.WebviewController();

@Builder
getContent() {
  Web({ src: '', controller: this.controller })
    .forceDisplayScrollBar(true)      // 强制显示滚动条
    .textZoomRatio(this.fontSizePct)
    .domStorageAccess(true)           // DOM Storage API
    .fileAccess(true)                 // 应用中文件系统的访问
    .imageAccess(true)                // 自动加载图片资源
    .javaScriptAccess(true)           // 执行JavaScript脚本
    .onControllerAttached(() => {
      this.controller.loadUrl(this.url);
    })
    .layoutWeight(1)
    .margin({ top: 8 });
}
```

**`textZoomRatio` is the whole feature**, and it is a percentage of the page's
authored sizes, not an absolute point size. Because it is an attribute bound to
`@State`, changing `fontSizePct` re-applies it in place: the page is not
reloaded, the scroll position is kept, and the reflow is immediate. That is the
advantage over injecting CSS through `runJavaScript`, which would need the page
to have finished loading and would be undone by an in-page navigation.

**`src: ''` plus `loadUrl` in `onControllerAttached` is a deliberate idiom, not
a workaround.** `WebviewController` methods raise an error if the controller is
not yet associated with a `Web` component, and `onControllerAttached` is the
callback that guarantees the association - it fires before page load begins, so
this is the earliest and safest place to start navigation. The alternative,
passing the URL as `src`, is fine too, but then any later programmatic
navigation still has to wait for this callback.

The four `*Access` switches deserve a second look before reuse. `fileAccess(true)`
grants the page access to the application's file system; combined with
`javaScriptAccess(true)` on a remote URL, that is a wider surface than a
read-only article needs. `domStorageAccess` is required only if the page uses
`localStorage`. Turn on what the content actually uses.

**Stepping the size — same file** (corrected, see `HW-11-0018`)

```typescript
Column() { /* the A+ icon and its label */ }
  .onClick(() => {
    if (this.fontSizePct === FontSizePreference.FONT_SIZE_LARGE) {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.fontsize_max') });
      return;
    } else {
      this.fontSizePct = this.fontSizePct + CommonConstants.FONT_SIZE_INTERVAL;
      this.dataPreferences?.putSync(CommonConstants.PREFERENCE_FONT_SIZE_NAME, this.fontSizePct);
      this.dataPreferences?.flush();   // FIX: absent in the sample - putSync is memory-only
    }
  })
  .onTouch((event: TouchEvent) => {
    if (event.type === TouchType.Down) {
      this.isPressedIncrease = true;
    } else if (event.type === TouchType.Up) {
      this.isPressedIncrease = false;
    }
  });
```

**`putSync` writes to the in-memory `Preferences` instance and nothing else.**
The reference is explicit for the XML store: "Data operations are performed in
the memory. To persist data, call flush()." Without it the value survives as
long as the process does and vanishes on a cold start - which is precisely the
capability the document's 场景介绍 claims ("实现字体大小设置的持久化存储",
persistent storage of the font-size setting). That is `HW-11-0018`. `flush()`
is asynchronous and returns a promise; `flushSync()` is the blocking form, and
either is acceptable on a control that fires at most a few times a minute.

**`getPromptAction()` from the `UIContext` is the current form of `showToast`.**
The global `promptAction.showToast` still exists but is not bound to a UI
instance, which misbehaves in multi-window and multi-instance apps. Reaching it
through `this.getUIContext()` inside a component is the sanctioned route.

The `onTouch` pair driving `isPressedIncrease` is the sample's way of getting a
pressed state on a `Column` that is not a `Button`. It is honest but leaky:
`TouchType.Cancel` is not handled, so a touch that turns into a scroll leaves
the icon stuck in its pressed image. Add `event.type === TouchType.Cancel` to
the reset branch, or use `.stateStyles({ pressed: ... })`.

**Restoring the setting — same file** (corrected, see `HW-11-0018`)

```typescript
dataPreferences: preferences.Preferences | null = null;

aboutToAppear(): void {
  let context = this.getUIContext().getHostContext();
  this.dataPreferences = preferences.getPreferencesSync(context, { name: 'myPreferences' });
  // FIX: the sample does putSync(key, FONT_SIZE_NORMAL) here, overwriting the
  // stored choice with the default on every entry. Read, do not write.
  this.fontSizePct = this.dataPreferences?.getSync(
    CommonConstants.PREFERENCE_FONT_SIZE_NAME,
    FontSizePreference.FONT_SIZE_NORMAL) as FontSizePreference;
}
```

**This is the second half of the same defect.** Even with `flush()` added to the
two step handlers, the shipped `aboutToAppear` resets the stored key to
`FONT_SIZE_NORMAL` before the user can benefit from it - the write path and the
read path are both missing, and the sample only appears to work within a single
visit because `@State` carries the value. `getSync` with the default as the
second argument is the whole restore: it returns the stored value when the key
exists and the default when it does not, so no "first run" branch is needed.

Note `getPreferencesSync` needs the ability context; `getHostContext()` returns
`Context | undefined`, so the nullable `dataPreferences` field and the `?.`
calls throughout are load-bearing, not defensive noise.

**Back navigation through the Web history — same file** (as shipped)

```typescript
onBackward() {
  if (this.controller.accessBackward()) {
    this.controller.backward();   // 返回上一个web页
    // 执行用户自定义返回逻辑
    return true;
  } else {
    // 执行系统默认返回逻辑，返回上一个page页
    return false;
  }
}
```

**`accessBackward()` before `backward()` is the required order** - calling
`backward()` with no history is an error, not a no-op. The boolean return is
shaped for `onBackPressed`: `true` means "handled here", `false` means "let the
system pop the page". In the sample it is wired only to the on-screen back
arrow in `topBar()`, so the hardware/gesture back still leaves the page even
when the Web has history. Wiring the same method into the page's back handler
is a two-line change and the obvious completion.

## Permissions & config

Declared in `entry/src/main/module.json5`:

```json5
"requestPermissions": [{
  "name": 'ohos.permission.INTERNET'
}]
```

- `INTERNET` is `system_grant` (normal), so no runtime request and no `reason`
  or `usedScene` are needed - which is why the entry has no other fields.
- It is genuinely required: the `Web` loads `https://developer.huawei.com` from
  the `target_url` string resource, and without the permission the load fails
  with a network error rather than a permission dialog.
- No storage permission is involved - `preferences` writes inside the app's own
  sandbox.

The page root is a `NavDestination` with `hideToolBar(true)` and a `@Provide`d
`NavPathStack`, but the `@Entry` component *is* that `NavDestination` - there is
no `Navigation` container above it. It renders, and the structure is there so
the page can be pushed into a real navigation stack later, but the provided
stack is never used and `pageInfo` has no consumer in this sample.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` in the zip is
  `6.0.0(20)`, matching the document. `textZoomRatio` itself is available from
  API 9.
- **The persistence does not work as shipped** - neither the write (no `flush`)
  nor the read (`aboutToAppear` writes the default instead of reading). Both
  fixes are above; the feature is otherwise complete.
- `textZoomRatio` scales text only. Images, fixed-width containers and anything
  sized in `px` in the page's CSS do not scale with it, so a page laid out with
  absolute widths will overflow at 150 percent. Pages using `rem`/`em` and
  fluid widths behave best.
- The document's 实现思路 snippet names the enum members
  `FontSizePreference.FontSizeLarge` / `FontSizeNormal`, while the sample
  declares `FONT_SIZE_LARGE` / `FONT_SIZE_NORMAL`. Copying the document's code
  into the sample does not compile.
- The collect and share buttons in the bottom bar have `onTouch` press states
  but no `onClick` - they are placeholders.
- `expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.BOTTOM])` on the bottom
  bar lets its background paint under the navigation indicator; the icons stay
  above it because the padding is applied to the `Row`'s content.
- The bottom bar is a fixed `Row` of four 50 vp columns with `SpaceBetween`; at
  the largest system display size the 10 vp labels will clip rather than wrap.

## Pitfalls

- **`HW-11-0018`** (B/medium, confirmed): the font size is `putSync`'d at three
  call sites (`WebPage.ets:37/120/148`) and no `flush()` or `flushSync()`
  exists anywhere in the sample, so the setting lives only in the in-memory
  `Preferences` instance and is silently lost when the process is killed -
  defeating the persistence the document's 场景介绍 explicitly claims. **Fix:**
  call `this.dataPreferences?.flush()` after each `putSync` (or once in
  `aboutToDisappear`). Fix the read side at the same time: `aboutToAppear`
  should `getSync` the stored value into `fontSizePct` instead of writing
  `FONT_SIZE_NORMAL` over it.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-attributes.md` - `textZoomRatio`, `javaScriptAccess`, `domStorageAccess`, `fileAccess`, `forceDisplayScrollBar`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-basic-components-web-attributes
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` - `getPreferencesSync`, `putSync`, `getSync`, `flush`/`flushSync` and the memory-only warning
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- `documentation/harmonyos-guides/03_application-framework/data-persistence-by-preferences.md` - the intended read/write/flush cycle
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/data-persistence-by-preferences
- `documentation/harmonyos-guides/03_application-framework/web-page-loading-with-web-components.md` - `WebviewController`, `onControllerAttached`, `loadUrl`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/web-page-loading-with-web-components
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.INTERNET`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `NEWS-14` - the same `preferences` API used correctly, with `flushSync` after every write
- `huawei_industry_tree/11_news_reading/docs/16_h5_fontsize.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5_fontsize-0000002289377882
