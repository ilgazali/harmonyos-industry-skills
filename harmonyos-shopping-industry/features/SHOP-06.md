---
id: SHOP-06
title: Pull-down to navigate - reuse Refresh as a gesture host that pushes a page instead of reloading one
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/06_pull_to_jump.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/pull_to_jump-0000002282295284
sample: huawei_industry_tree/16_shopping/downloads/PullToJump.zip
kits: ["@kit.ArkUI", "@kit.ArkWeb", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.CameraKit", "@kit.CoreFileKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [Refresh, refreshOffset, pullToRefresh, onStateChange, onOffsetChange, onRefreshing, RefreshStatus, "NavPathStack.pushPath", NavDestination, Navigation, Tabs, onContentWillChange, Web, onShowFileSelector, "cameraPicker.pick", "photoAccessHelper.PhotoViewPicker", "picker.DocumentViewPicker", "@StorageProp", "window.getWindowAvoidArea"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-16-0027, HW-16-0029, HW-16-0030, HW-16-0031]
status: verified
---

## When to use

Load this card when a page needs a **pull-down gesture that goes somewhere**
rather than one that reloads. Shopping apps use it for the customer-service
shortcut on the cart page; the same shape covers pull-to-open-search,
pull-to-show-order-status, or the "second floor" pattern (see `SHOP-22` for the
full second-floor variant).

The trick is that `Refresh` is not really a refresh component — it is a
pull-distance gesture host with a threshold, a state machine and a slot for
your own header. Nothing forces the `onRefreshing` callback to fetch data. Here
it immediately sets `refreshing` back to `false` (so the built-in spinner never
appears) and pushes a `NavDestination` instead. You get the platform's rubber
banding, the threshold hysteresis and the header-height animation for free,
without writing a pan gesture.

Adopt it when the destination is *one* fixed target and the gesture should feel
like part of the scroll. If the pull needs several outcomes depending on
distance, or the header must stay on screen, write a `PanGesture` — `Refresh`
gives you exactly one trigger point.

## Feature checklist

- A five-tab shopping shell; only the cart tab (index 2) is implemented and
  reachable.
- Pulling down on the cart tab reveals a custom header: a customer-service icon
  above a hint line.
- The hint reads 下拉跳转 (pull to jump) while the pull is short and switches to
  松开跳转 (release to jump) once it passes the trigger offset.
- Releasing past the threshold pushes the 客服中心 (service centre) page.
- No refresh spinner ever shows — the refreshing state is cancelled in the same
  frame it is entered.
- The service page hosts a local `Web` page whose file inputs raise the photo
  picker, camera picker or document picker according to what the page asks for.
- Back on the service page pops the navigation stack and returns to the cart.

## Architecture

One `entry` module, three pages, one constants file.

```
entry/src/main/ets
├── common/Constants.ets                every literal in the sample, ~70 statics
├── entryability/EntryAbility.ets       full-screen window, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages
│   ├── BuyingCarPage.ets               the cart tab's content (search bar, scan row, A–Z list)
│   ├── MainPage.ets                    @Entry: Navigation > Tabs > Refresh > BuyingCarPage
│   └── ServicePage.ets                 the push target: NavDestination wrapping a Web view
└── utils/Logger.ets                    hilog wrapper
entry/src/main/resources/base/profile/route_map.json   'ServicePage' -> ServicePageBuilder
entry/src/main/resources/rawfile/WebPicker.html        the local page the Web view loads
```

The documented tree matches the zip.

**The design decision worth copying** is the layering:
`Navigation` (owns the stack) → `Tabs` (owns the shell) → `Refresh` (owns the
gesture) → the page content. Each of the three outer components does one job
and none of them knows about the others' state. In particular `Refresh` sits
*inside* the `TabContent`, not around the `Tabs` — so the gesture belongs to
one tab, and the tab bar never moves when the user pulls.

The `NavPathStack` is a plain field on `MainPage` (`pageInfos: NavPathStack = new NavPathStack()`),
handed to `Navigation(this.pageInfos)`, and the destination is resolved through
`module.json5`'s `routerMap` rather than a `navDestination` builder attribute.
That is the form to prefer: `pushPath({ name: 'ServicePage' })` needs no import
of the target page, so the pushing page and the pushed page stay decoupled and
the destination is lazily loaded. The target declares its own entry with an
exported `@Builder` function:

```typescript
@Builder
export function ServicePageBuilder() {
  ServicePage();
}
```

matched by `route_map.json`'s `buildFunction: "ServicePageBuilder"`.

## Implementation steps

1. **Put `Refresh` inside the `TabContent`,** wrapping only the scrollable
   content of the tab that owns the gesture.
2. **Bind `refreshing` with `$$`** — `Refresh({ refreshing: $$this.isRefreshing, … })`.
   The two-way binding is required: the component writes the flag when the
   gesture enters the refreshing state, and you write it back to cancel.
3. **Set the trigger distance with `refreshOffset`.** The default is 64 vp
   (96 vp when `promptText` is set); this sample uses 100 vp, a deliberately
   longer pull because the outcome is a navigation, not a reload.
4. **Enable `pullToRefresh(true)`** so exceeding `refreshOffset` on release
   actually fires. It defaults to `true`, but stating it next to `refreshOffset`
   documents the pair.
5. **Supply a `builder`** for the pull area. With a custom builder the built-in
   spinner and `promptText` are both suppressed — the reference is explicit that
   `promptText` is ignored when `builder` is used.
6. **Constrain the builder's minimum height** (`constraintSize({ minHeight: 32 })`)
   and `clip(true)` it. The pull area's height tracks the drag, so without a
   floor the icon and text get squeezed to nothing at small offsets.
7. **Switch the hint on the state change.** `RefreshStatus.OverDrag` is the
   state "pulled past the threshold, release to trigger"; `Drag` is "pulled but
   not far enough". Compare against the enum, not the numeric literal the
   sample uses.
8. **Cancel the refreshing state and push in `onRefreshing`:** set
   `isRefreshing = false` first, then `pushPath`. Order matters — leave the flag
   true and the pull area stays extended behind the pushed page and is still
   there when the user pops back.
9. **Register the destination in `routerMap`,** not with a `navDestination`
   builder, and give the destination an `onReady` that captures its own
   `pathStack` so its back button can `pop()`.

## Verified snippets

All snippets are from `PullToJump.zip`. The one correction below is a
robustness fix, not a defect we filed.

**The gesture host — `entry/src/main/ets/pages/MainPage.ets`** (corrected — magic number replaced by the enum)

```typescript
TabContent() {
  Column() {
    Refresh({ refreshing: $$this.isRefreshing, builder: this.customRefreshComponent() }) {
      BuyingCarPage();
    }
    .pullToRefresh(true)
    .refreshOffset(Constants.REFRESH_OFFSET)          // 100 vp
    .onStateChange((refreshStatus: RefreshStatus) => {
      if (refreshStatus === RefreshStatus.OverDrag) { // FIX: shipped code compares to the literal 2
        this.jumpSign = $r('app.string.release');     // 松开跳转
      } else {
        this.jumpSign = $r('app.string.pulling');     // 下拉跳转
      }
    })
    .onOffsetChange((offset: number) => {
      this.pullOffset = offset;
    })
    .onRefreshing(() => {
      this.isRefreshing = false;                      // cancel before pushing: no spinner, no stuck header
      this.pageInfos.pushPath({ name: 'ServicePage' });
    });
  }
  .width(Constants.FULL)
  .height(Constants.FULL);
}.tabBar(this.BuildTabs($r('app.media.car'), $r('app.string.select_car'), 2));
```

**Three lines carry the whole feature.** `refreshOffset(100)` sets how far the
user must pull before the gesture commits — longer than a reload's default 64 vp
because an accidental navigation is more annoying than an accidental reload.
`onStateChange` gives the hysteresis for free: the component tracks `Inactive →
Drag → OverDrag → Refresh` itself, and `OverDrag` (value 2) means "past the
threshold, release now". The reference spells out the transitions — from
`OverDrag`, releasing enters `Refresh`, swiping back up returns to `Drag` — which
is exactly the two-way hint behaviour without any distance arithmetic on your
side.

`onRefreshing` is where the component stops being a refresh. Setting
`isRefreshing = false` synchronously means the `Refresh` state machine leaves
the refreshing state in the same frame it entered it, so the header collapses
and no loading indicator is drawn. Then the push happens. Reverse those two
statements and the pull area animates open under the new page.

The shipped comparison `refreshStatus === 2` works but is brittle: it is a
public enum with named members, and `RefreshStatus.OverDrag` says what the
branch means.

**The custom pull area — same file** (as shipped)

```typescript
@Builder
customRefreshComponent() {
  Stack() {
    Column() {
      Image($r('app.media.service_icon'))
        .width(Constants.SERVICE_ICON_WIDTH);
      Text(this.jumpSign)
        .fontSize(Constants.JUMP_SING_FONTSIZE)
        .margin({ top: Constants.JUMP_SING_MARGIN_TOP });
    }
    .alignItems(HorizontalAlign.Center);
  }
  .align(Alignment.Center)
  .clip(true)
  // 设置最小高度约束保证自定义组件高度随刷新区域高度变化时自定义组件高度不会低于minHeight。
  .constraintSize({ minHeight: Constants.MIN_HEIGHT })   // 32 vp
  .width(Constants.FULL);
}
```

**`clip(true)` plus `constraintSize({ minHeight })` is the pair that makes a
custom header behave.** The refreshing area's height is driven by the drag
distance, and the builder is laid out inside it. Without the minimum height the
content collapses and the icon disappears at the start of the pull; without the
clip the content overflows the area and paints over the page while the header
is shorter than its content. The sample's own comment says exactly this. The
outer `Stack` with `.align(Alignment.Center)` keeps the icon and hint centred
in whatever height the area currently has.

`this.jumpSign` is a `Resource`, not a `string`, so the two hint texts are
translatable and the state handler swaps resource references rather than
literals.

**The push target — `entry/src/main/ets/pages/ServicePage.ets`** (as shipped)

```typescript
@Builder
export function ServicePageBuilder() {
  ServicePage();
}

@Component
export struct ServicePage {
  pathStack: NavPathStack = new NavPathStack();
  controller: webview.WebviewController = new webview.WebviewController();

  build() {
    NavDestination() {
      Column() {
        Row() {
          Image($r('app.media.back_icon'))
            .onClick(() => {
              this.pathStack.pop();
            });
          Text($r('app.string.title'));            // 客服中心
        }
        .padding({ top: this.getUIContext().px2vp(this.topRectHeight) });

        Web({ src: $rawfile(Constants.WEB_PAGE), controller: this.controller })
          .zoomAccess(false)
          .javaScriptAccess(true)
          .domStorageAccess(true)
          .fileAccess(false)
          .geolocationAccess(false)
          .onShowFileSelector((event) => { /* routes to photo / camera / document picker */ });
      }
      .padding({ bottom: this.getUIContext().px2vp(this.bottomRectHeight) });
    }
    .hideTitleBar(true)
    .title('ServicePage')
    .onReady((context: NavDestinationContext) => {
      this.pathStack = context.pathStack;           // the stack that actually pushed us
    });
  }
}
```

**`onReady` is what makes the back button work.** The field initialiser
`new NavPathStack()` is a placeholder: popping it would do nothing, because it
is not the stack `MainPage` owns. `NavDestinationContext.pathStack` hands back
the real stack at attach time, and the assignment in `onReady` replaces the
placeholder before any user interaction can happen. Skipping this and importing
the parent's stack instead is the coupling the `routerMap` indirection exists to
avoid.

The `Web` settings are the other half worth copying: `fileAccess(false)` and
`geolocationAccess(false)` are switched off explicitly even though the page is a
local `$rawfile`, and `javaScriptAccess(true)` is on only because
`onShowFileSelector` calls `runJavaScript('getFileType()')` back into the page to
decide which picker to raise. That handler returns `true`, which tells the web
engine the app has taken over the file choice — returning `false` would let the
default selector run as well.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

The three pickers it uses are all *picker* APIs — `photoAccessHelper.PhotoViewPicker`,
`cameraPicker.pick` and `picker.DocumentViewPicker` — which run in a system UI
and hand back URIs the app is temporarily authorised for. That is the reason no
`READ_IMAGEVIDEO`, `CAMERA` or storage permission appears anywhere: choosing the
picker form of each API is what removes the permission declaration. Copy that
choice, not just the code.

```json5
"pages": "$profile:main_pages",
"routerMap": "$profile:route_map"
```

```json5
// route_map.json
{ "name": "ServicePage",
  "pageSourceFile": "src/main/ets/pages/ServicePage.ets",
  "buildFunction": "ServicePageBuilder" }
```

`deviceTypes`: `phone`, `tablet`, `2in1`. `main_pages` lists `pages/MainPage`
only — `ServicePage` is reached exclusively through the router map.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- `refreshOffset` and `pullToRefresh` are API 12+; `builder` in `RefreshOptions`
  is likewise a v12 addition. On older SDKs only the default spinner exists.
- **`promptText` is dead once you pass a `builder`.** The reference states the
  prompt text will not be displayed when `builder` or `refreshingContent`
  customises the area.
- `onContentWillChange` returns `false` for every `comingIndex !== 2`, so four
  of the five bottom tabs are intentionally inert — the tab bar highlight does
  not even move, because `onChange` never fires. This is a deliberate demo
  restriction here (the other tabs have empty `TabContent` bodies), but it is
  the same construct that silently breaks navigation when copied without the
  guard; see `TOUR-03`'s `HW-09-0016` for the accidental version.
- `pullOffset` is written by `onOffsetChange` on every frame of the drag and
  never read. Either drive something with it (a rotating icon, a progressive
  hint) or drop the handler — a per-frame `@State` write that nothing consumes
  still triggers re-render bookkeeping.
- `Tabs` is `.scrollable(false)`, so the only way between tabs is the bar.
- `BuyingCarPage`'s A–Z index strip is decorative: `selectedIndex` is a plain
  field and no handler updates it.
- `Constants.PHONE_WIDTH` is captured once from `display.getDefaultDisplaySync().width`
  at class-load time, so it does not follow a window resize on 2in1.

## Pitfalls

No defects were filed against this document or sample during review. Two
observations that did not rise to findings are recorded under Constraints: the
numeric `refreshStatus === 2` comparison (corrected in the snippet above) and
the unused `pullOffset` state.

The systematic doc-snippet defect `HW-16-0013` does **not** apply here — this
document's single snippet is abridged with `// ...` inside the block bodies and
remains syntactically valid, which is exactly the form the fix for that finding
prescribes.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-refresh.md` — `RefreshOptions`, `refreshOffset`, `pullToRefresh`, `onStateChange`/`onOffsetChange`/`onRefreshing`, and the `RefreshStatus` transition table
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-refresh
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` — `Navigation`, `NavPathStack.pushPath`, `NavDestination` and `NavDestinationContext`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` — the `routerMap` form of destination registration
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `SHOP-22` — the second-floor pattern, the heavier alternative when the pull target is a full screen of its own
- `huawei_industry_tree/16_shopping/docs/06_pull_to_jump.md` — the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/pull_to_jump-0000002282295284
