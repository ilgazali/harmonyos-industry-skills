---
id: COMMON-34
title: Launch another app from an H5 page by URL scheme - intercept the navigation, check the link, fall back to AppGallery
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/34_app_pull_up.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_pull_up-0000002353615821
sample: huawei_industry_tree/19_common_technical_solutions/downloads/AppPullUp.zip
kits: ["@kit.ArkWeb", "@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["Web.onLoadIntercept", "OnLoadInterceptEvent.data", "WebResourceRequest.getRequestUrl", "bundleManager.canOpenLink", "UIAbilityContext.startAbility", Want, querySchemes, skills, "PromptAction.showDialog", "promptAction.ShowDialogSuccessResponse", "PromptAction.showToast", "Web.fileAccess", "Web.geolocationAccess", "$rawfile", Tabs, TabContent]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0102, HW-19-0103, HW-19-0104, HW-19-0105, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when an embedded H5 page contains links in a **custom URL scheme**
that should open another application - a "open in the Tips app" banner, a
partner-app handoff, a payment app launch - and the host application must handle
the case where that app is not installed.

Three parties are involved and all three need configuration: the **page** emits
the scheme URL, the **launching application** declares which schemes it may query
and intercepts the navigation, and the **target application** declares that it
can be opened by that scheme.

## Feature checklist

The launching application must:

- Declare every scheme it will ever query in `module.json5`'s `querySchemes`.
- Intercept navigations with `Web.onLoadIntercept` and read the request URL.
- Distinguish scheme URLs from ordinary page navigations, and let the latter
  through.
- Check the link with `bundleManager.canOpenLink`, inside `try/catch`
  (HW-19-0102).
- Launch with `startAbility({ uri })` when it can be opened, and report a launch
  failure.
- Offer an AppGallery download when it cannot, targeting the **right** bundle
  (HW-19-0104), and only when the user actually confirms (HW-19-0103).

The target application must:

- Declare a `skills` entry with `ohos.want.action.viewData` and a matching
  `uris.scheme`.

## Architecture

Single-module project (`entry` HAP), one page:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup; publishes `topRectHeight` / `bottomRectHeight` |
| `pages/WebPage.ets` | a five-tab shell whose last tab hosts the `Web` component and the whole interception |
| `resources/rawfile/pullApp.html` | the page with the scheme links, plus `style.css` and a `picture/` asset folder |

**The configuration is symmetric and both halves are required.**

Launching side - `AppPullUp.zip#AppPullUp/entry/src/main/module.json5`:

```json5
"querySchemes": [
  "hwtips",
],
```

Target side, as the document specifies:

```json5
"skills": [
  {
    "actions": [
      "ohos.want.action.viewData"
    ],
    "uris": [
      { "scheme": "hwtips" }
    ]
  }
]
```

Without `querySchemes`, `canOpenLink` throws 17700056 rather than returning
`false` - which is what makes HW-19-0102 matter.

**Interception flow.**

```
page navigates to hwtips://...
  -> onLoadIntercept(event)
       url = event.data.getRequestUrl()
       is it http/https/resource?  -> fall through (return false at the end)
       otherwise:
         canOpenLink(url)
           true  -> startAbility({ uri: url })
                      .then  -> launched
                      .catch -> "launch failed" dialog
           false -> "download?" dialog -> startAbility({ uri: 'store://appgallery...' })
```

The scheme test is a **negative** filter - anything that is not http, https or
resource is treated as a launchable link. That is the root of two of the four
findings: it hands `canOpenLink` schemes the module never declared
(HW-19-0102), and it routes every one of them to the same hardcoded store page
(HW-19-0104).

**`onLoadIntercept`'s return value** decides whether the navigation proceeds:
returning `true` blocks it (the application handled it), `false` lets the web
view continue. The sample returns `true` inside the scheme branch and `false`
otherwise.

## Implementation steps

1. **Declare the schemes** you will query in `querySchemes`, and keep that list
   in step with whatever maps a scheme to a bundle (HW-19-0104).
2. **Have the target application declare the skill** - `ohos.want.action.viewData`
   plus the `uris.scheme` entry.
3. **Load the page** and lock the web view down: `.fileAccess(false)`,
   `.geolocationAccess(false)`.
4. **Intercept**: `onLoadIntercept((event) => { const url =
   event.data.getRequestUrl(); ... })`.
5. **Classify the URL.** Prefer a positive test - does the scheme appear in your
   declared list - over the negative "not http/https/resource" test the sample
   uses.
6. **Check inside `try/catch`** and treat a throw as "cannot open"
   (HW-19-0102).
7. **Launch** with `startAbility({ uri: url })`, handling both the promise
   rejection and the callback error.
8. **Fall back** to the store, inspecting the dialog response before acting
   (HW-19-0103) and building the store URI from the scheme's own bundle
   (HW-19-0104).
9. **Return `true`** from the branch you handled, `false` otherwise.

## Verified snippets

All snippets below come from the sample project, not from the document.

### The whole interception

`AppPullUp.zip#AppPullUp/entry/src/main/ets/pages/WebPage.ets`

```ts
Web({ src: $rawfile('pullApp.html'), controller: this.controller })
  .height($r('app.string.web_height'))
  .fileAccess(false)
  .geolocationAccess(false)
  .onLoadIntercept((event) => {
    if (event) {
      let url: string = event.data.getRequestUrl();
      hilog.info(0x0000, 'URL', '链接为' + url);
      if ((url.indexOf('http://') !== 0) && (url.indexOf('https://') !== 0) &&
        (url.indexOf('resource://') !== 0)) {
        let canOpen = bundleManager.canOpenLink(url);        // FIX (HW-19-0102): wrap in try/catch
        if (canOpen) {
          let want: Want = {
            uri: url
          };
          let context = this.getUIContext().getHostContext() as common.UIAbilityContext;
          context.startAbility(want).then(() => {
            //拉起成功
          }).catch(() => {
            this.promptAction.showDialog({
              title: $r('app.string.tips'),
              message: $r('app.string.fail'),
              buttons: [{
                text: $r('app.string.confirm'),
                color: $r('app.string.fontcolor'),
              }]
            });
          });
        } else {
          this.promptAction.showDialog({
            title: $r('app.string.tips'),
            message: $r('app.string.download'),
            buttons: [{
              text: $r('app.string.confirm'),
              color: $r('app.string.fontcolor'),
            }]
          })
            .then(() => {                                    // FIX (HW-19-0103): check result.index
              let want: Want = {
                uri: `store://appgallery.huawei.com/app/detail?id=com.huawei.hmos.tips`   // FIX (HW-19-0104)
              };
              let context = this.getUIContext().getHostContext() as common.UIAbilityContext;
              context.startAbility(want, (err: BusinessError) => {
                if (err.code) {
                  console.error(`startAbility failed, code is ${err.code}, message is ${err.message}`);
                  return;
                }
                console.info('startAbility succeed');
              });
            });
        }
      }
    }
    return false;
  });
```

Corrected check and fallback:

```ts
let canOpen = false;
try {
  canOpen = bundleManager.canOpenLink(url);
} catch (err) {
  hilog.error(0x0000, 'URL',
    `canOpenLink failed. Code: ${(err as BusinessError).code}, message: ${(err as BusinessError).message}`);
}
if (canOpen) {
  // startAbility({ uri: url })
} else {
  const bundleName = SCHEME_TO_BUNDLE.get(url.split(':')[0]);
  if (!bundleName) {
    return true;
  }
  this.promptAction.showDialog({ /* ... */ })
    .then((result: promptAction.ShowDialogSuccessResponse) => {
      if (result.index !== 0) {
        return;
      }
      context.startAbility({ uri: `store://appgallery.huawei.com/app/detail?id=${bundleName}` }, cb);
    });
}
```

### The scheme declaration

`AppPullUp.zip#AppPullUp/entry/src/main/module.json5`

```json5
{
  "module": {
    "querySchemes": [
      "hwtips",
    ],
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "requestPermissions":[
      {
        "name": "ohos.permission.INTERNET"    // FIX (HW-19-0105): not needed for a rawfile page
      },
    ],
    "deliveryWithInstall": true,
    "installationFree": false,
    "pages": "$profile:main_pages",
    // ... abilities ...
  }
}
```

### The tab shell

`AppPullUp.zip#AppPullUp/entry/src/main/ets/pages/WebPage.ets`

```ts
@Builder
tabBuilder(title: ResourceStr, selectImg: ResourceStr) {
  Column() {
    Image(selectImg)
      .width($r('app.string.tab_img_width'));
    Text(title)
      .fontSize($r('app.float.tab_text_fontsize'));
  }
  .margin({ bottom: $r('app.string.tab_column_margin_bottom') })
  .onClick(() => {
    this.promptAction.showToast({
      message: '该功能待开发'
    });
  });
}

// ... Tabs({ barPosition: BarPosition.End, index: 4 }) with four empty TabContents
// and the Web component in the fifth.
```

The first four tabs are placeholders that toast 该功能待开发 ("this feature is not
yet developed") - only the fifth tab carries the feature.

## Permissions & config

**The feature itself needs no permission.** `bundleManager.canOpenLink` requires
only the `querySchemes` declaration - not a permission - and `startAbility` with
a `uri` Want needs none either. The sample nevertheless declares
`ohos.permission.INTERNET` for a page that loads entirely from `rawfile`
(HW-19-0105).

What **is** required is configuration on both sides:

- Launching side: `querySchemes` listing every scheme passed to `canOpenLink`.
- Target side: a `skills` entry with `ohos.want.action.viewData` and a matching
  `uris.scheme`, so the system can resolve the Want.

`module.json5` also declares `"deviceTypes": ["phone", "tablet", "2in1"]` and the
usual `EntryAbility` / `EntryBackupAbility` pair.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. `bundleManager.canOpenLink` is an
  API 12 interface.
- **`canOpenLink` requires the scheme to be declared**: "The scheme specified in
  the link must be configured in the **querySchemes** field of the module.json5
  file", and it throws 17700056 otherwise (and 17700055 for an invalid link).
- **`querySchemes` is a privacy boundary, not a convenience.** It is what stops an
  application from probing which apps are installed, so it cannot be widened at
  runtime - the list has to be known at build time.
- **The target must opt in.** Without the `viewData` skill and the scheme in
  `uris`, the target application is not reachable no matter what the launcher
  declares.
- **`onLoadIntercept` returns a boolean.** `true` blocks the navigation, `false`
  lets it continue; the sample's outer `return false` is what allows normal page
  navigation to work.
- **`showDialog` resolves on dismissal too**, not only on a button press.
- **Testing needs the target application installed**, and the AppGallery fallback
  needs a device with AppGallery.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **`canOpenLink` is called unguarded, which is incorrect.** The negative scheme
  filter admits every non-http URL - `mailto:`, `tel:`, `about:`, anything
  malformed - while only `hwtips` is declared, so the documented 17700056 and
  17700055 throws escape the interceptor. The reference's own example uses
  `try/catch`. (HW-19-0102)
- **`.then(() => ...)` on `showDialog` is incorrect.** The dialog resolves when it
  closes, so tapping outside it still launches AppGallery. Inspect
  `ShowDialogSuccessResponse.index`. (HW-19-0103)
- **The store URI hardcodes `com.huawei.hmos.tips`, which is incorrect for a
  generic launcher.** Any scheme that fails the check sends the user to the Tips
  app's store page. Derive the bundle from the scheme. (HW-19-0104)
- **`ohos.permission.INTERNET` is declared for a page that loads no network
  resources.** (HW-19-0105)
- **The scheme test is negative, not positive.** `!http && !https && !resource`
  will keep admitting new URL forms as the page evolves; testing the scheme
  against the declared `querySchemes` list is both safer and self-documenting.
- **The interceptor returns `false` after handling a scheme link.** The handled
  branch does not return `true`, so the web view is also told to proceed with a
  navigation the application just consumed - harmless for an unresolvable scheme,
  but it is not what the boolean is for.
- **`getRequestUrl` returns whatever the page asked for.** It is untrusted input:
  it is used to build a `Want.uri` that starts another application, so validate
  the scheme before passing it on rather than after.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-bundlemanager.md` -
  `bundleManager.canOpenLink`, the `querySchemes` precondition, error codes
  17700055 / 17700056, and the `try/catch` example.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-bundlemanager
- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-events.md` -
  `onLoadIntercept` and `WebResourceRequest.getRequestUrl`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-basic-components-web-events#onloadintercept10
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/deep-linking-startup -
  the Deep Linking flow, `querySchemes` and the target-side `skills`
  configuration.
- `documentation/harmonyos-guides/01_getting-started/module-configuration-file.md` -
  `querySchemes`, `skills`, `actions` and `uris`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/module-configuration-file
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-promptaction.md` -
  `showDialog` and `ShowDialogSuccessResponse`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-promptaction
- `documentation/harmonyos-guides/03_application-framework/web-component-overview.md` -
  when `ohos.permission.INTERNET` is actually required.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/web-component-overview
- `documentation/harmonyos-guides/04_system/app-permission-mgmt-overview.md` -
  "Request only the least required permissions for your application."
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/app-permission-mgmt-overview
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_pull_up-0000002353615821
