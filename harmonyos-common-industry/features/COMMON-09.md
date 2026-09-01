---
id: COMMON-09
title: H5 side-swipe back interception - consume the back gesture inside the web history before leaving the page
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/09_backpage_by_gesture.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/backpage_by_gesture-0000002237371200
sample: huawei_industry_tree/19_common_technical_solutions/downloads/BackPageByGestures.zip
kits: ["@kit.ArkWeb", "@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: ["NavDestination.onBackPressed", "webview.WebviewController", "WebviewController.accessStep", "WebviewController.backward", "Web.fileAccess", "Web.geolocationAccess", "$rawfile", Navigation, NavPathStack, "NavPathStack.pushPath", routerMap, "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.on('avoidAreaChange')", "UIContext.px2vp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0013, HW-19-0014, HW-19-0015, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when an application embeds an **H5 page with internal navigation**
and the side-swipe back gesture must behave the way the user expects: go back one
step **inside the web page** if there is one, and only leave the native page when
the web history is exhausted.

Without this, the first back swipe pops the whole `NavDestination` and the user
loses their place in the web content - the exact complaint the document names:
"用户浏览H5页面时需要侧滑回退至上一页而非框架页面上一页" ("when the user browses an
H5 page, the side swipe should go back to the previous web page, not to the
previous page of the native framework").

## Feature checklist

The application must:

- Host the `Web` component inside a `NavDestination`.
- Intercept the back gesture with `NavDestination.onBackPressed`.
- Ask the `WebviewController` whether one backward step is possible before
  consuming the gesture.
- Go back inside the web view and return `true` when it is.
- Return `false` when it is not, so the framework performs its own back
  navigation.
- Fail safe: if the check throws, fall through to framework back navigation
  rather than leaving the gesture unhandled (HW-19-0014).
- Restrict the web view's surface: no file-system access, no geolocation, when
  the content does not need them.
- Declare `ohos.permission.INTERNET` **only** if online pages are loaded
  (HW-19-0013).

## Architecture

Single-module project (`entry` HAP), two pages plus route table:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | loads `pages/Index`, sets full-screen layout, publishes and keeps updated the status-bar and navigation-indicator insets in `AppStorage` |
| `pages/Index.ets` | the native framework page: teenage-mode consent screen with a checkbox that enables the button; owns the `NavPathStack` |
| `pages/WebPage.ets` | the `NavDestination` that hosts the `Web` component and implements the interception |
| `resources/base/profile/route_map.json` | maps route name `WebPage` to `webPageBuilder` in `WebPage.ets` |
| `resources/rawfile/TeenageAgreement.html`, `SoftwareLicense.html` | the two local pages that give the web view a history to go back through |

Navigation and interception flow:

1. `Index` builds a `Navigation(this.pathStack)` and, on button click, calls
   `this.pathStack.pushPath({ name: 'WebPage' })`.
2. The name resolves through `module.json5`'s `"routerMap": "$profile:route_map"`
   to the exported `@Builder webPageBuilder()`, which constructs the `WebPage`
   component - so `WebPage` is not registered in code, only in the profile.
3. `WebPage` renders `NavDestination { Web({ src: $rawfile('TeenageAgreement.html'),
   controller }) }`.
4. The user taps the in-page link to `SoftwareLicense.html`; the web view now has
   two history entries.
5. A back swipe fires `NavDestination.onBackPressed`. `accessStep(-1)` returns
   `true`, so `backward()` runs and the callback returns `true` - the gesture is
   consumed and the `NavDestination` stays.
6. On the first page `accessStep(-1)` returns `false`, the callback returns
   `false`, and the framework pops back to `Index`.

The contract that makes this work is `onBackPressed`'s boolean return: `true`
means "handled, do not pop"; `false` means "not handled, perform the default".

## Implementation steps

1. **Set up the route table.** Add `"routerMap": "$profile:route_map"` to
   `module.json5` and a `route_map.json` entry naming the page, its source file
   and its builder function. Export the builder from the page file with
   `@Builder export function webPageBuilder() { WebPage(); }`.
2. **Own one `NavPathStack`** in the entry page and wrap the content in
   `Navigation(this.pathStack)`; push with `pushPath({ name: 'WebPage' })`.
3. **Root the web page in a `NavDestination`** - `onBackPressed` is a
   `NavDestination` attribute, so a page routed any other way cannot intercept
   the gesture this way.
4. **Create the controller** as a member of the page component:
   `controller: WebviewController = new webview.WebviewController()`.
5. **Mount the `Web` component** with the controller and lock down what it may
   reach: `.fileAccess(false)` and `.geolocationAccess(false)` for local
   content. Note that `fileAccess` "does not affect the access to the files
   specified through `$rawfile`", so rawfile-relative links and images keep
   working with it disabled.
6. **Implement the interception** in `onBackPressed`: `accessStep(-1)` decides,
   `backward()` acts, `true` consumes, `false` delegates. Wrap the calls in
   `try/catch` and return `false` from the catch (HW-19-0014).
7. **Declare permissions honestly.** Add `ohos.permission.INTERNET` only if the
   web view loads online pages (HW-19-0013).
8. **Handle the full-screen insets.** Read `TYPE_SYSTEM` and
   `TYPE_NAVIGATION_INDICATOR` avoid areas, publish them to `AppStorage`,
   subscribe to `'avoidAreaChange'`, and consume them as page padding via
   `@StorageProp` + `px2vp`.

## Verified snippets

All snippets below come from the sample project, not from the document.

### The interception

`BackPageByGestures.zip#BackPageByGestures/entry/src/main/ets/pages/WebPage.ets`

```ts
import { webview } from '@kit.ArkWeb';

@Builder
export function webPageBuilder() {
  WebPage();
}

@Component
struct WebPage {
  @StorageProp('topRectHeight')
  topRectHeight: number = 0;
  @StorageProp('bottomRectHeight')
  bottomRectHeight: number = 0;
  controller: WebviewController = new webview.WebviewController;

  build() {
    NavDestination() {
      Web({ src: $rawfile('TeenageAgreement.html'), controller: this.controller })
        .fileAccess(false)
        .geolocationAccess(false)
        .height('100%');
    }
    .padding({
      top: this.getUIContext().px2vp(this.topRectHeight),
      bottom: this.getUIContext().px2vp(this.bottomRectHeight)
    })
    .hideTitleBar(true)
    .onBackPressed(() => {
      if (this.controller.accessStep(-1)) {
        this.controller.backward(); // 返回上一页
        return true;
      } else {
        return false;
      }
    });
  }
}
```

Corrected `onBackPressed` (HW-19-0014):

```ts
.onBackPressed(() => {
  try {
    if (this.controller.accessStep(-1)) {
      this.controller.backward();
      return true;
    }
  } catch (error) {
    hilog.error(0x0000, 'WebPage',
      `accessStep failed. Code: ${(error as BusinessError).code}, message: ${(error as BusinessError).message}`);
  }
  return false;
})
```

### Route registration

`BackPageByGestures.zip#BackPageByGestures/entry/src/main/resources/base/profile/route_map.json`

```json
{
  "routerMap": [
    {
      "name": "WebPage",
      "pageSourceFile": "src/main/ets/pages/WebPage.ets",
      "buildFunction": "webPageBuilder"
    }
  ]
}
```

### Pushing the route

`BackPageByGestures.zip#BackPageByGestures/entry/src/main/ets/pages/Index.ets`

```ts
@Entry
@Component
struct BackPageByGesturesPage {
  pathStack: NavPathStack = new NavPathStack();
  @StorageProp('topRectHeight') topRectHeight: number = 0;
  @StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
  @State isOpen: boolean = false;

  build() {
    Navigation(this.pathStack) {
      Column() {
        // ... consent copy ...
        Toggle({ type: ToggleType.Checkbox })
          .onChange((isOn: boolean) => {
            this.isOpen = isOn;
          });

        Button($r('app.string.button_text'))
          .enabled(this.isOpen)
          .onClick(() => {
            this.pathStack.pushPath({ name: 'WebPage' });
          });
      }
      .padding({
        top: this.getUIContext().px2vp(this.topRectHeight),
        bottom: this.getUIContext().px2vp(this.bottomRectHeight)
      });
    }
    .hideToolBar(true);
  }
}
```

### Insets, kept current

`BackPageByGestures.zip#BackPageByGestures/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
let typeTop = window.AvoidAreaType.TYPE_SYSTEM; // 状态栏避让
let typeBottom = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR; // 导航条避让
let avoidAreaTop = windowClass.getWindowAvoidArea(typeTop);
let avoidAreaBottom = windowClass.getWindowAvoidArea(typeBottom);
AppStorage.setOrCreate('topRectHeight', avoidAreaTop.topRect.height);
AppStorage.setOrCreate('bottomRectHeight', avoidAreaBottom.bottomRect.height);
windowClass.on('avoidAreaChange', (data) => {
  if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
    AppStorage.setOrCreate('topRectHeight', data.area.topRect.height);
  } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
    AppStorage.setOrCreate('bottomRectHeight', data.area.bottomRect.height);
  }
});
```

## Permissions & config

**For this sample: no permission is needed.** Every page it loads is a packaged
`$rawfile`. The shipped `module.json5` nevertheless declares
`ohos.permission.INTERNET` (HW-19-0013) - drop it unless your web view loads
online pages, in which case keep it, because the official Web overview states
"The **ohos.permission.INTERNET** permission is required for accessing online web
pages."

`BackPageByGestures.zip#BackPageByGestures/entry/src/main/module.json5` - as
shipped:

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "deliveryWithInstall": true,
    "installationFree": false,
    "pages": "$profile:main_pages",
    "requestPermissions": [
      { "name": "ohos.permission.INTERNET" }   // remove for local-only content (HW-19-0013)
    ],
    "routerMap": "$profile:route_map",         // required by pushPath({ name: 'WebPage' })
    "abilities": [
      {
        "name": "EntryAbility",
        "srcEntry": "./ets/entryability/EntryAbility.ets",
        "icon": "$media:layered_image",
        "label": "$string:EntryAbility_label",
        "startWindowIcon": "$media:startIcon",
        "startWindowBackground": "$color:start_window_background",
        "exported": true,
        "skills": [
          { "entities": ["entity.system.home"], "actions": ["action.system.home"] }
        ]
      }
    ],
    "extensionAbilities": [
      {
        "name": "EntryBackupAbility",
        "srcEntry": "./ets/entrybackupability/EntryBackupAbility.ets",
        "type": "backup",
        "exported": false,
        "metadata": [
          { "name": "ohos.extension.backup", "resource": "$profile:backup_config" }
        ]
      }
    ]
  }
}
```

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`onBackPressed` is a `NavDestination` attribute.** The page must be routed
  through Navigation; a Router-based page cannot use this interception point.
- **`accessStep` can throw**: 17100001 "Init error. The WebviewController must be
  associated with a Web component." and 401 parameter error.
- **`fileAccess` default changed at API 12.** "For API version 11 and earlier
  versions, access to the file system in the application is enabled by default if
  this attribute is not explicitly called. Since API version 12, access to the
  file system in the application is disabled by default." Setting it to `false`
  explicitly, as the sample does, is the safe form either way - and it "does not
  affect the access to the files specified through `$rawfile`".
- **`setCustomUserAgent` interacts with history.** The official reference warns
  that calling it during the first load can leave `accessBackward` false even with
  multiple history entries; set the user agent before `loadUrl` if you use it.
- **Devices.** `phone`, `tablet`, `2in1` per `module.json5`; `accessStep` itself
  is available on Phone, PC/2in1, Tablet, TV and Wearable.

## Pitfalls

- **The document says the network permission is required, which is incorrect for
  this sample.** Every page is a `$rawfile`; `ohos.permission.INTERNET` is
  required only "for accessing online web pages", and the official permission
  principles say to "Request only the least required permissions ... Do not apply
  for unnecessary or obsolete permissions." (HW-19-0013)
- **Calling `accessStep` unguarded is incorrect.** The official reference example
  wraps it in `try/catch`; an exception escaping `onBackPressed` means the
  callback returns no boolean and the back gesture is left in an undefined state.
  Catch, log, and return `false`. (HW-19-0014)
- **The documented project tree is incomplete, which is incorrect.** It omits
  `entrybackupability/EntryBackupAbility.ets` and, more importantly,
  `resources/base/profile/route_map.json`; without the route map,
  `pushPath({ name: 'WebPage' })` resolves to nothing. (HW-19-0015)
- **Do not read `accessStep(-1)` as "can go back at all".** It asks whether a
  specific number of steps can be taken; `-1` is one step backward. For the
  simple case `accessBackward()` is the more direct API.
- **The web view needs a history for this feature to be observable.** The sample
  ships two rawfile pages linked to each other for exactly that reason; a single
  static page will always take the `false` branch.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-webview-webviewcontroller.md` -
  `accessStep(step: number): boolean`, its error codes, and the official
  `try/catch` example; `backward()`; the `setCustomUserAgent` history caveat.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-webview-webviewcontroller
- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-attributes.md` -
  `fileAccess`, its API 12 default change, and the `$rawfile` exemption.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-basic-components-web-attributes
- `documentation/harmonyos-guides/03_application-framework/web-component-overview.md` -
  "The ohos.permission.INTERNET permission is required for accessing online web
  pages."
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/web-component-overview
- `documentation/harmonyos-guides/03_application-framework/web-page-loading-with-web-components.md` -
  network, local and rich-text page loading.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/web-page-loading-with-web-components
- `documentation/harmonyos-guides/04_system/app-permission-mgmt-overview.md` -
  "Request only the least required permissions for your application."
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/app-permission-mgmt-overview
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-e.md` -
  `AvoidAreaType.TYPE_SYSTEM` and `TYPE_NAVIGATION_INDICATOR`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-e
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-basic-components-navdestination -
  `onBackPressed` and its boolean contract.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/backpage_by_gesture-0000002237371200
