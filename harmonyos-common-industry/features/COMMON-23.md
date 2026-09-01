---
id: COMMON-23
title: H5 scan integration - expose Scan Kit to a web page through a JavaScript proxy bridge
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/23_web_scan.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_scan-0000002284631808
sample: huawei_industry_tree/19_common_technical_solutions/downloads/WebScanDemo.zip
kits: ["@kit.ScanKit", "@kit.ArkWeb", "@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["scanBarcode.startScanForResult", "scanBarcode.ScanOptions", "scanBarcode.ScanResult", "scanCore.ScanType", "abilityAccessCtrl.createAtManager", "AtManager.requestPermissionsFromUser", "WebviewController.registerJavaScriptProxy", "WebviewController.deleteJavaScriptRegister", "WebviewController.setPathAllowingUniversalAccess", "WebviewController.runJavaScript", "WebviewController.loadUrl", "WebviewController.refresh", "WebviewController.accessBackward", "WebviewController.backward", "webview.WebviewController.setWebDebuggingAccess", "Web.onControllerAttached", "Web.onPageEnd", "Web.javaScriptAccess", "Web.fileAccess", "component .expandSafeArea", "Page.onBackPress"]
permissions: ["ohos.permission.CAMERA", "ohos.permission.INTERNET"]
min_api: 20
modules: [entry]
findings: [HW-19-0073, HW-19-0074, HW-19-0075, HW-19-0076, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when an **H5 page inside the application needs to scan a QR code
or barcode** - a payment page, a coupon redemption flow, a membership card - and
the scanning must use the native Scan Kit camera UI rather than a JavaScript
decoder in the page.

The pattern is a JSBridge: a native object is injected into the page's `window`,
the page calls its methods, and the native side runs the permission request and
the scanner.

**Read HW-19-0073 before reusing.** The bridge as shipped stays registered after
the web view navigates to a remote site, which hands that site the camera entry
point.

## Feature checklist

The application must:

- Load the H5 page (a packaged `$rawfile` in the sample).
- Register a native object into the page as a JavaScript proxy, listing exactly
  the methods the page may call.
- **Delete that registration** when leaving the trusted page, and at the latest
  on page disappear (HW-19-0073).
- Request `ohos.permission.CAMERA` at runtime and check `authResults` before
  scanning.
- Launch the default scanner UI with `scanBarcode.startScanForResult`.
- Return a result the page can classify unambiguously - success, failure, denial
  (HW-19-0075).
- Handle the back gesture by walking the web history first.
- Keep web debugging out of release builds (HW-19-0074).

## Architecture

Single-module project (`entry` HAP), three source files:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup; loads `pages/WebScan` |
| `pages/WebScan.ets` | the `Web` component, the proxy registration, the cross-origin path allowance, the immersive-fit script and the back-gesture handling |
| `utils/Scanner.ets` | the bridged object: `scan()` (permission + Scan Kit) and `handleScanResult()` (act on the decoded value) |
| `resources/rawfile/index.html` | the H5 page that calls `window.nativeScanner.*` |

**The bridge.** In `onControllerAttached` - the earliest point at which the
controller is usable - the page constructs a `Scanner`, hands it the controller,
the ability `Context` (needed for the permission request and for
`startScanForResult`) and the `UIContext` (needed for the toast), then:

```ts
this.webVC.registerJavaScriptProxy(this.scannerObj, 'nativeScanner', ['scan', 'handleScanResult']);
this.webVC.setPathAllowingUniversalAccess([this.context.resourceDir]);
this.webVC.refresh();
```

The `refresh()` is required: a proxy registered after the page has loaded is not
visible to it until the page is reloaded.

**Round trip.** The page calls `nativeScanner.scan()`, which returns a promise
resolving to a string; the page passes that string back to
`nativeScanner.handleScanResult(value)`, and the native side decides what to do -
in the sample, navigate the web view to a target URL.

**Two contexts, two purposes.** `Context` (the ability context) is what
`requestPermissionsFromUser` and `startScanForResult` need;
`UIContext` is what `getPromptAction()` needs. The sample passes both into the
`Scanner` constructor rather than reaching for globals.

**Immersive fitting.** `expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.TOP,
SafeAreaEdge.BOTTOM])` lets the web content run under the system bars, and an
`onPageEnd` script injects padding into the remote page's DOM to compensate. That
script is tightly coupled to the target site's class names (`r-145dblm`) and is
demo scaffolding, not a reusable technique.

## Implementation steps

1. **Declare both permissions.** `ohos.permission.CAMERA` (user-grant, with
   `reason` and `usedScene`) and `ohos.permission.INTERNET`.
2. **Load the local page** with `Web({ src: $rawfile('index.html'), controller })`
   and enable `javaScriptAccess(true)`.
3. **Register the proxy in `onControllerAttached`**, list only the methods the
   page needs, then `refresh()` so the page sees it.
4. **Allow the cross-origin paths the page actually needs**:
   `setPathAllowingUniversalAccess([this.context.resourceDir])`.
5. **Request the camera permission inside `scan()`** and check
   `permission.authResults[0]` before continuing - the sample's own comment
   flags that it only implements the success branch.
6. **Build `ScanOptions`** - the sample uses `scanTypes: [scanCore.ScanType.ALL]`,
   `enableMultiMode: true`, `enableAlbum: true` - and `await
   scanBarcode.startScanForResult(this.context, options)` inside `try/catch`.
7. **Return a structured result**, not a bare string (HW-19-0075).
8. **Delete the registration** before navigating away from the trusted page and
   in `aboutToDisappear` (HW-19-0073).
9. **Handle back**: `onBackPress` -> `accessBackward()` -> `backward()`, return
   `true` when consumed.

## Verified snippets

All snippets below come from the sample project, not from the document.

### The bridged scanner

`WebScanDemo.zip#WebScanDemo/entry/src/main/ets/utils/Scanner.ets`

```ts
import { scanBarcode, scanCore } from '@kit.ScanKit';
import abilityAccessCtrl from '@ohos.abilityAccessCtrl';
import { webview } from '@kit.ArkWeb';
import { hilog } from '@kit.PerformanceAnalysisKit';

const GET_CAMERA_PERMISSION_ALLOWED: number = 0;
const NO_CAMERA_PERMISSION: string = 'No camera permission';

export default class Scanner {
  private webVC: webview.WebviewController | undefined;
  private context: Context;
  private uiContext: UIContext;

  constructor(webviewController: webview.WebviewController, context: Context, uiContext: UIContext) {
    if (webviewController) {
      this.webVC = webviewController;
    }
    this.context = context;
    this.uiContext = uiContext;
  }

  /**
   * 调用Scan Kit扫码二维码
   */
  async scan(): Promise<string> {
    // 权限申请
    let atManager = abilityAccessCtrl.createAtManager();
    let permission = await atManager.requestPermissionsFromUser(this.context, ['ohos.permission.CAMERA']);

    // 本Demo只实现了权限申请成功示例，可针对权限申请结果做不同校验
    if (permission.authResults[0] === GET_CAMERA_PERMISSION_ALLOWED) {
      // 定义扫码参数options
      let options: scanBarcode.ScanOptions = {
        scanTypes: [scanCore.ScanType.ALL],
        enableMultiMode: true,
        enableAlbum: true
      };

      // 调用Scan Kit能力完成扫码并返回结果
      try {
        let result: scanBarcode.ScanResult = await scanBarcode.startScanForResult(this.context, options);
        let scanResult: string = result.originalValue;
        return scanResult;
      } catch (error) {
        let errMsg: string = `errCode: ${error.code}, errMsg: ${error.message}`;
        hilog.error(DOMAIN, 'testTag', 'Failed to get ScanResult by promise with options. Cause: %{public}s', errMsg);
        return errMsg;
      }
    }
    return NO_CAMERA_PERMISSION;
  }

  /**
   * 扫码结果处理，根据实际业务调整
   */
  handleScanResult(scanResult: string) {
    if (scanResult.search('errCode') >= 0) {         // FIX (HW-19-0075): use a status field
      hilog.info(DOMAIN, 'testTag', '%{public}', scanResult);   // FIX (HW-19-0076): '%{public}s'
    } else if (scanResult === NO_CAMERA_PERMISSION) {
      hilog.info(DOMAIN, 'testTag', '%{public}', NO_CAMERA_PERMISSION);
      this.uiContext.getPromptAction().showToast({
        message: $r('app.string.no_permission')
      });
    } else {
      this.webVC?.loadUrl($r('app.string.test_target_url'));
    }
  }
}
```

### Registering the bridge

`WebScanDemo.zip#WebScanDemo/entry/src/main/ets/pages/WebScan.ets`

```ts
@Entry
@Component
struct WebScan {
  private webVC: webview.WebviewController = new webview.WebviewController();
  private scannerObj: Scanner | undefined;
  private context = this.getUIContext().getHostContext() as Context;
  private uiContext = this.getUIContext();

  aboutToAppear(): void {
    webview.WebviewController.setWebDebuggingAccess(true);   // FIX (HW-19-0074)
  }

  build() {
    Column() {
      Web({
        src: $rawfile('index.html'),
        controller: this.webVC
      })
        .javaScriptAccess(true)
        .geolocationAccess(false)
        .domStorageAccess(true)
        .imageAccess(true)
        .fileAccess(true)
        .imageAccess(true)     // FIX (HW-19-0076): duplicated
        .fileAccess(true)      // FIX (HW-19-0076): duplicated
        .expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.TOP, SafeAreaEdge.BOTTOM])
        .onControllerAttached(() => {
          try {
            // 初始化JSBridge
            this.scannerObj = new Scanner(this.webVC, this.context, this.uiContext);
            this.webVC.registerJavaScriptProxy(this.scannerObj, 'nativeScanner', ['scan', 'handleScanResult']);
            // FIX (HW-19-0073): pair with deleteJavaScriptRegister('nativeScanner')
            // 设置允许可以跨域访问的路径列表
            this.webVC.setPathAllowingUniversalAccess([this.context.resourceDir]);

            this.webVC.refresh();
          } catch (error) {
            hilog.error(DOMAIN, 'testTag', 'Failed to initialize JSBridge. Cause: %{public}s',
              `ErrorCode: ${(error as BusinessError).code},  Message: ${(error as BusinessError).message}`);
          }
        });
    }
    .height('100%')
    .width('100%');
  }
}
```

### Back-gesture handling

`WebScanDemo.zip#WebScanDemo/entry/src/main/ets/pages/WebScan.ets`

```ts
onBackPress(): boolean | void {
  try {
    // 侧滑返回上一页
    let result = this.webVC.accessBackward();
    if (result) {
      this.webVC.backward();
      return true;
    }
    return false;
  } catch (error) {
    hilog.error(DOMAIN, 'testTag', 'Failed to backward. Cause: %{public}s',
      `ErrorCode: ${(error as BusinessError).code},  Message: ${(error as BusinessError).message}`);
    return false;
  }
}
```

This is the `Page.onBackPress` variant of the same interception COMMON-09
implements with `NavDestination.onBackPressed`, and unlike that sample it does
wrap the call in `try/catch` and return `false` from the catch.

### The navigation target

`WebScanDemo.zip#WebScanDemo/entry/src/main/resources/base/element/string.json`

```json
{
  "name": "test_target_url",
  "value": "https://www.vmall.com/"
}
```

## Permissions & config

Both permissions the document lists are required: `ohos.permission.CAMERA` for
the scanner and `ohos.permission.INTERNET` because the sample navigates to a
remote page after a successful scan.

`WebScanDemo.zip#WebScanDemo/entry/src/main/module.json5`:

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.CAMERA",
    "reason": "$string:reason_for_camera",
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": "inuse"
    }
  },
  {
    "name": "ohos.permission.INTERNET"
  }
]
```

`ohos.permission.CAMERA` is **user-grant**, so the declaration is not enough -
`scan()` requests it at runtime on every invocation and checks
`permission.authResults[0]`. `ohos.permission.INTERNET` is system-grant and needs
only the declaration.

Note the contrast with COMMON-15, which drives the camera through
`cameraPicker.pick` and therefore needs no camera permission at all:
`scanBarcode.startScanForResult` opens the scanner inside the application's own
permission scope.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`registerJavaScriptProxy` is controller-scoped, not page-scoped.** "After
  registerJavaScriptProxy is called, the application exposes the registered
  JavaScript object to all page frames" - including frames of a page loaded
  later.
- **It must be paired with `deleteJavaScriptRegister`**: "The
  registerJavaScriptProxy API must be used together with the
  deleteJavaScriptRegister API to prevent memory leak."
- **Only for trusted URLs**: "It is recommended that registerJavaScriptProxy be
  used only with trusted URLs and over secure HTTPS connections. Injecting
  JavaScript objects into untrusted web components can expose your application to
  malicious attacks."
- **A method must appear in exactly one list**: "You should register
  registerJavaScriptProxy either in synchronous list or in asynchronous list.
  Otherwise, this API fails to be registered", and a method in both "is called
  asynchronously by default".
- **`refresh()` after registration** - the page must reload to see the injected
  object.
- **`setWebDebuggingAccess` defaults to `false`** and should stay so in release.
- **The `onPageEnd` DOM script targets a specific remote site** (`react-root`,
  `r-145dblm`); it is demo scaffolding and will silently do nothing elsewhere.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **The proxy is never deleted, which is incorrect.** After
  `handleScanResult` navigates to `https://www.vmall.com/`, that remote page and
  every frame in it can call `nativeScanner.scan()` - raising the camera
  permission dialog and launching the scanner - and `nativeScanner.handleScanResult()`
  to drive the web view. Delete the registration before leaving the packaged
  page. (HW-19-0073)
- **`setWebDebuggingAccess(true)` in `aboutToAppear` is incorrect for a released
  app**, and here it exposes a bridge that reaches the camera. (HW-19-0074)
- **Discriminating the scan outcome with `scanResult.search('errCode')` is
  incorrect.** The searched string is attacker-chosen barcode content, so a code
  containing that substring is dropped as an error; return a status field
  instead. (HW-19-0075)
- **`'%{public}'` is not a valid format identifier** - it has no type letter, so
  the value is not printed - and `imageAccess` / `fileAccess` are each set twice
  on the `Web` component. (HW-19-0076)
- **`scan()` requests the permission on every call.** That is harmless once
  granted, but it means the page can trigger a system dialog at will - one more
  reason the bridge must not outlive the trusted page.
- **`authResults[0] === 0` is checked, but the denied branch only returns a
  string.** The sample's own comment admits this
  ("本Demo只实现了权限申请成功示例", "this demo only implements the
  permission-granted example"); a production flow should distinguish
  "denied once" from "denied permanently" and route the user to settings.
- **`fileAccess(true)` is deliberate here.** It is needed together with
  `setPathAllowingUniversalAccess([resourceDir])`; do not copy it into a page
  that does not need local file access.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-webview-webviewcontroller.md` -
  `registerJavaScriptProxy` and its four NOTE constraints,
  `deleteJavaScriptRegister`, `setPathAllowingUniversalAccess`, `runJavaScript`,
  `loadUrl`, `accessBackward` / `backward`, and `setWebDebuggingAccess` (default
  `false`, security note).
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-webview-webviewcontroller
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/scan-scanbarcode-api -
  `scanBarcode.startScanForResult`, `ScanOptions`, `ScanResult.originalValue`.
- `documentation/harmonyos-guides/03_application-framework/web-in-page-app-function-invoking.md` -
  the JSBridge pattern the reference points to for a full example.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/web-in-page-app-function-invoking
- `documentation/harmonyos-guides/04_system/request-user-authorization.md` -
  requesting a user-grant permission and checking `authResults`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/request-user-authorization
- `documentation/harmonyos-references/03_system/js-apis-hilog.md` - format
  identifier syntax and the argument-mapping rule.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/camera-preparation -
  camera permission preparation, linked by the document.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_scan-0000002284631808
