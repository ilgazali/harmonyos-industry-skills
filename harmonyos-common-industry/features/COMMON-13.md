---
id: COMMON-13
title: User identity authentication - fingerprint or lock-screen PIN gate in front of a confidential document preview
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/13_user_authentication.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/user_authentication-0000002290451165
sample: huawei_industry_tree/19_common_technical_solutions/downloads/UserAuthDemo.zip
kits: ["@kit.UserAuthenticationKit", "@kit.PreviewKit", "@kit.CoreFileKit", "@kit.BasicServicesKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.CryptoArchitectureKit"]
apis: ["userAuth.getUserAuthInstance", "userAuth.AuthParam", "userAuth.WidgetParam", "userAuth.UserAuthType", "userAuth.AuthTrustLevel", "userAuth.UserAuthResultCode", "UserAuthInstance.on('result')", "UserAuthInstance.off('result')", "UserAuthInstance.start", "cryptoFramework.createRandom", "Random.generateRandomSync", "fileUri.getUriFromPath", "filePreview.canPreview", "filePreview.openPreview", "request.downloadFile", "DownloadTask.on('complete')", "DownloadTask.off('complete')", "fileIo.accessSync", NavDestination, NavPathStack, "NavDestination.onReady", "component .bindSheet", SheetType, routerMap]
permissions: ["ohos.permission.ACCESS_BIOMETRIC", "ohos.permission.INTERNET"]
min_api: 20
modules: [entry]
findings: [HW-19-0025, HW-19-0026, HW-19-0027, HW-19-0028, HW-19-0029]
status: verified-with-fixes
---

## When to use

Load this card when a screen or an asset must be **behind a device-credential
gate**: opening a confidential document, entering a banking area, confirming a
payment. The user proves who they are with the same credential the device
unlocks with - fingerprint, or lock-screen PIN as fallback - and only then does
the protected content load.

This is device-local identity verification through User Authentication Kit; it is
not a login and it produces no account session. It does produce an
authentication **token**, which is what a server would verify if the gate has to
be trusted beyond the device.

## Feature checklist

The application must:

- Offer fingerprint authentication as the primary path.
- Offer lock-screen PIN as an alternative path, reachable from a "more options"
  sheet.
- Build an `AuthParam` with a **freshly generated random challenge** per attempt
  (HW-19-0025), the wanted `authType`, and an appropriate `authTrustLevel`.
- Build a `WidgetParam` whose `title` states the purpose of the authentication.
- Subscribe to `'result'`, start the authentication, and **unsubscribe when it
  completes** (HW-19-0026).
- Distinguish success from every other `UserAuthResultCode` and tell the user
  what happened (HW-19-0027).
- On success, download the document if it is not cached and open it in the system
  file preview.
- Declare `ohos.permission.ACCESS_BIOMETRIC` (required by the authentication APIs)
  and `ohos.permission.INTERNET` (required by the download).

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup, publishes insets, loads `pages/MainPage` |
| `common/Constants.ets` | `CommonConstant.FILE_NAME` and `WEB_URL`, the https URL of the sample PDF |
| `pages/MainPage.ets` | the approval home; pushes the route `userAuthPage` |
| `pages/UserAuthPage.ets` | the gate: fingerprint button, "more options" sheet with the PIN and cancel buttons, and the pop-on-success callback |
| `utils/UserAuth.ets` | `startUserAuth` (shared), `pullUpLock` (PIN), `fingerIdentification` (fingerprint) |
| `utils/FileUtil.ets` | `filePre` (download-if-absent, then preview) and `viewFile` (`canPreview` then `openPreview`) |
| `utils/Logger.ets` | thin `hilog` wrapper |
| `component/CommonTimeLine.ets`, `FileComponent.ets`, `PdfComponent.ets` | approval timeline and file list UI |
| `resources/base/profile/route_map.json` | route `userAuthPage` -> `UserAuthPageBuilder` |

**Control flow.**

1. `MainPage` pushes `userAuthPage`.
2. Tapping the fingerprint image calls `UserAuth.fingerIdentification(callback,
   context)`; the "more options" sheet's password button calls
   `UserAuth.pullUpLock(...)`. The two differ only in `authType`
   (`FINGERPRINT` vs `PIN`), `authTrustLevel` (`ATL1` vs `ATL2`) and the dialog
   title.
3. Both funnel into `UserAuth.startUserAuth`, which creates a `UserAuthInstance`,
   subscribes to `'result'`, and calls `start()`. The system draws the
   authentication dialog - the document notes it cannot even be screen-recorded.
4. On `UserAuthResultCode.SUCCESS` the callback calls `filePre(context)` and then
   the page's `authCallback`, which pops the gate off the navigation stack.
5. `filePre` checks the cache directory with `fs.accessSync`; if the PDF is
   absent it downloads it with `request.downloadFile` and previews it from the
   `'complete'` event, otherwise it previews immediately.
6. `viewFile` converts the sandbox path to a URI with `fileUri.getUriFromPath`,
   asks `filePreview.canPreview`, and calls `filePreview.openPreview` with a
   `PreviewInfo` carrying the title, URI and `mimeType`.

**Where the security actually lives.** The gate as written is a UI decision: the
success branch simply navigates. `UserAuthResult.token` is never read. For a
gate that a server must trust, the challenge has to come from the server, be
random per request (HW-19-0025), and the returned token has to be sent back for
verification.

## Implementation steps

1. **Declare the permissions** in `module.json5`:
   `ohos.permission.ACCESS_BIOMETRIC` (the reference marks the authentication APIs
   "**Required permissions**: ohos.permission.ACCESS_BIOMETRIC") and
   `ohos.permission.INTERNET` for the download. Both are system_grant, so there is
   no runtime dialog.
2. **Generate a random challenge** per attempt:
   ```ts
   const rand = cryptoFramework.createRandom();
   const randData = rand?.generateRandomSync(16)?.data;
   if (!randData) { return; }
   ```
   Do not reuse a constant (HW-19-0025). Maximum 32 bytes.
3. **Build `AuthParam`**: `{ challenge: randData, authType: [...],
   authTrustLevel: ... }`.
4. **Build `WidgetParam`**: `title` is mandatory, must be non-empty and at most
   500 characters, and the reference advises setting it to the authentication
   purpose ("such as payment or application login"). Resolve it from resources
   with `resourceManager.getStringSync($r('app.string.x').id)`.
5. **Create the instance, subscribe, start**, all inside `try/catch`:
   `getUserAuthInstance(authParam, widgetParam)`, `on('result', { onResult })`,
   `start()`.
6. **Handle every result code**, not just `SUCCESS` (HW-19-0027), and call
   `off('result')` on the same instance when done (HW-19-0026).
7. **Log the launch failure properly** in the `catch` - resolve resources, use
   `error` level (HW-19-0029).
8. **On success, load the protected content.** Guard both promises with
   rejection handlers and release the download listener (HW-19-0028).
9. **Preview through PreviewKit**: sandbox path -> `fileUri.getUriFromPath` ->
   `canPreview` -> `openPreview({ title, uri, mimeType })`.

## Verified snippets

All snippets below come from the sample project, not from the document.

### The shared authentication launcher

`UserAuthDemo.zip#UserAuthDemo/entry/src/main/ets/utils/UserAuth.ets`

```ts
import { userAuth } from '@kit.UserAuthenticationKit';
import Logger from './Logger';
import { filePre } from './FileUtil';
import { common } from '@kit.AbilityKit';

export class UserAuth {
  // 拉起用户认证
  static async startUserAuth(authParam: userAuth.AuthParam, widgetParam: userAuth.WidgetParam,
    callback: VoidCallback, context: common.Context) {
    try {
      let userAuthInstance = userAuth.getUserAuthInstance(authParam, widgetParam);
      userAuthInstance.on('result', {
        onResult: (result) => {
          // 认证成功
          if (result.result === userAuth.UserAuthResultCode.SUCCESS) {
            filePre(context);
            callback();
            Logger.info('testTag', 'userAuthInstance callback result ' + JSON.stringify(result));
          }
          // FIX (HW-19-0027): else branch reporting result.result
          // FIX (HW-19-0026): userAuthInstance.off('result');
        }
      });
      userAuthInstance.start();
    } catch (e) {
      // FIX (HW-19-0029): resolve the resource, use Logger.error
      Logger.info(`${$r('app.string.pull_up_fail')} ${e.code} ${e.message}`);
    }
  }
}
```

### The two authentication parameter sets (as shipped - see HW-19-0025)

`UserAuthDemo.zip#UserAuthDemo/entry/src/main/ets/utils/UserAuth.ets`

```ts
// 拉起系统锁屏密码验证
static async pullUpLock(callback: VoidCallback, context: common.Context) {
  let widTitle: string = context.resourceManager.getStringSync($r('app.string.input_password').id);
  let authParam: userAuth.AuthParam = {
    challenge: new Uint8Array([49, 49, 49, 49, 49, 49]), // FIX: generate randomly per request
    authType: [userAuth.UserAuthType.PIN],
    authTrustLevel: userAuth.AuthTrustLevel.ATL2,
  };
  let widgetParam: userAuth.WidgetParam = {
    title: widTitle
  };
  UserAuth.startUserAuth(authParam, widgetParam, callback, context);
}

//拉起指纹验证
static async fingerIdentification(callback: VoidCallback, context: common.Context) {
  let widTitle: string = context.resourceManager.getStringSync($r('app.string.please_verify_the_fingerprint').id);
  let authParam: userAuth.AuthParam = {
    challenge: new Uint8Array([49, 49, 49, 49, 49, 49]), // FIX: generate randomly per request
    authType: [userAuth.UserAuthType.FINGERPRINT],
    authTrustLevel: userAuth.AuthTrustLevel.ATL1,
  };
  let widgetParam: userAuth.WidgetParam = {
    title: widTitle
  };
  UserAuth.startUserAuth(authParam, widgetParam, callback, context);
}
```

Corrected challenge generation, following the official reference example:

```ts
import { cryptoFramework } from '@kit.CryptoArchitectureKit';

function newChallenge(): Uint8Array | null {
  const rand = cryptoFramework.createRandom();
  const len: number = 16;
  let randData: Uint8Array | null = null;
  let retryCount = 0;
  while (retryCount < 3) {
    randData = rand?.generateRandomSync(len)?.data;
    if (randData) { break; }
    retryCount++;
  }
  return randData;
}
```

### The gate page

`UserAuthDemo.zip#UserAuthDemo/entry/src/main/ets/pages/UserAuthPage.ets`

```ts
@Builder
export function UserAuthPageBuilder() {
  UserAuthPage();
}

@Component
struct UserAuthPage {
  @State navPathStack: NavPathStack = new NavPathStack();
  @State context: Context = this.getUIContext().getHostContext() as common.Context;
  @State isShow: Boolean = false;
  @StorageProp('topRectHeight') topRectHeight: number = 0;

  @Builder
  myBuilder() {
    Column({ space: 16 }) {
      Button($r('app.string.password'))
        .onClick(() => {
          UserAuth.pullUpLock(this.authCallback, this.context);
          this.isShow = false;
        });
      Button($r('app.string.cancel'))
        .onClick(() => {
          this.isShow = false;
        });
    };
  }

  build() {
    NavDestination() {
      // ... header ...
      Column() {
        Image($r('app.media.finger_clock'))
          .onClick(() => {
            UserAuth.fingerIdentification(this.authCallback, this.context);
          });
        Text($r('app.string.click_to_unlock'));
        Text($r('app.string.more_options'))
          .onClick(() => {
            this.isShow = true;
          })
          .bindSheet($$this.isShow, this.myBuilder(), {
            height: 212,
            showClose: true,
            preferType: SheetType.CENTER,
            blurStyle: BlurStyle.COMPONENT_THICK,
            backgroundColor: '#F3F3F3',
            title: { title: '' }
          });
      };
    }
    .onReady((context: NavDestinationContext) => {
      this.navPathStack = context.pathStack;
    })
    .hideTitleBar(true);
  }

  private authCallback = () => {
    this.navPathStack.pop();
  };
}
```

### Download-if-absent, then preview

`UserAuthDemo.zip#UserAuthDemo/entry/src/main/ets/utils/FileUtil.ets`

```ts
//文件预览
export function viewFile(fileName: string, filePath: string, context: Context) {
  // 将沙箱路径转换为uri
  let uri = fileUri.getUriFromPath(filePath);
  filePreview.canPreview(context, uri).then((result) => {
    // true:可以预览
    if (result === true) {
      let fileInfo: filePreview.PreviewInfo = {
        title: fileName,
        uri: uri,
        mimeType: 'application/pdf'
      };
      filePreview.openPreview(context, fileInfo).then(() => {
      }).catch((err: BusinessError) => {   // FIX (HW-19-0028): log err
      });
    }
  }).catch((err: BusinessError) => {       // FIX (HW-19-0028): log err
  });
}

//预览文件，如之前未进行下载则先下载再预览。
export function filePre(context: common.Context) {
  let fileName: string = CommonConstant.FILE_NAME;
  let cacheDir = context.cacheDir;
  let filePath: string = cacheDir + '/' + fileName;
  let url: string = WEB_URL;

  try {
    //判断是否已下载过文件
    let res = fs.accessSync(filePath);
    if (res) {
      viewFile(fileName, filePath, context);
    } else {
      request.downloadFile(context, {
        url: url,
        filePath: filePath,
      }).then((downloadTask: request.DownloadTask) => {
        let completeCallback = () => {
          Logger.info('Download task completed.');
          viewFile(fileName, filePath, context);
          // FIX (HW-19-0028): downloadTask.off('complete', completeCallback);
        };
        downloadTask.on('complete', completeCallback);
      });
      // FIX (HW-19-0028): .catch((err: BusinessError) => Logger.error(...))
    }
  } catch (error) {
    let err: BusinessError = error as BusinessError;
    Logger.info(`Download failed: ${err.code} ${err.message}`);
  }
}
```

### Route table

`UserAuthDemo.zip#UserAuthDemo/entry/src/main/resources/base/profile/route_map.json`

```json
{
  "routerMap": [
    {
      "name": "userAuthPage",
      "pageSourceFile": "src/main/ets/pages/UserAuthPage.ets",
      "buildFunction": "UserAuthPageBuilder",
      "data": { "description": "this is UserAuthPage" }
    }
  ]
}
```

## Permissions & config

Both permissions the document lists are genuinely required here:
`ohos.permission.ACCESS_BIOMETRIC` because the reference marks the user
authentication APIs "**Required permissions**: ohos.permission.ACCESS_BIOMETRIC",
and `ohos.permission.INTERNET` because `request.downloadFile` fetches the PDF over
https.

`UserAuthDemo.zip#UserAuthDemo/entry/src/main/module.json5` - as shipped:

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "requestPermissions": [
      {
        "name": "ohos.permission.INTERNET",
        "reason": "$string:EntryAbility_desc",
        "usedScene": {
          "abilities": ["EntryAbility"],
          "when": "always"
        }
      },
      {
        "name": "ohos.permission.ACCESS_BIOMETRIC"
      }
    ],
    "deviceTypes": ["phone", "tablet", "2in1"],
    "routerMap": "$profile:route_map",
    "deliveryWithInstall": true,
    "installationFree": false,
    "pages": "$profile:main_pages",
    "abilities": [
      {
        "name": "EntryAbility",
        "srcEntry": "./ets/entryability/EntryAbility.ets",
        "icon": "$media:startIcon",
        "label": "$string:EntryAbility_label",
        "startWindowIcon": "$media:startIcon",
        "startWindowBackground": "$color:start_window_background",
        "exported": true,
        "skills": [
          { "entities": ["entity.system.home"], "actions": ["action.system.home"] }
        ]
      }
    ]
  }
}
```

The remote document URL lives in
`UserAuthDemo.zip#UserAuthDemo/entry/src/main/ets/common/Constants.ets` and is
`https://` - no cleartext transport.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **Devices.** `UserAuthInstance`, `on('result')` / `off('result')` and
  `UserAuthResult` are documented for **Phone, PC/2in1, Tablet, Wearable** - TV is
  not in the list. The sample declares `phone`, `tablet`, `2in1`.
- **`challenge` is at most 32 bytes** and is documented as a *random* value
  guarding against replay.
- **`WidgetParam.title`** cannot be empty and cannot exceed 500 characters.
- **`off('result')` must be called on the subscribing instance**: "The
  UserAuthInstance instance used to invoke this API must be the one used to
  subscribe to the event."
- **The authentication UI cannot be screen-recorded** - the document says so
  directly: "指纹认证和锁屏密码认证界面无法录制。" ("The fingerprint and lock-screen
  password authentication screens cannot be recorded.") Plan demos and automated
  UI tests accordingly.
- **The trust levels differ per path in this sample**: `ATL1` for fingerprint,
  `ATL2` for PIN. Choose the level from the sensitivity of what is being gated,
  not by copying the sample.
- **`WidgetParam.uiContext`** (API 18+) makes the dialog an application-modal
  dialog, but "This mode is applicable only to 2-in-1 devices."

## Pitfalls

- **The hardcoded `challenge: new Uint8Array([49, 49, 49, 49, 49, 49])` is
  incorrect.** The reference defines the field as a "Random challenge value, which
  can be used to prevent replay attacks", and every official example generates it
  with `cryptoFramework.createRandom()`. Generate it per request. (HW-19-0025)
- **Subscribing without ever calling `off('result')` is incorrect.** A new
  `UserAuthInstance` is created on every tap and only ever subscribed;
  unsubscribe when the result arrives. (HW-19-0026)
- **Handling only `UserAuthResultCode.SUCCESS` is incorrect.** Failure, lockout
  and cancellation all arrive through the same callback with a non-SUCCESS code
  and are currently dropped, leaving the user on an apparently unchanged screen.
  (HW-19-0027)
- **The preview path swallows every asynchronous failure, which is incorrect.**
  Two empty `catch` blocks, a `downloadFile` promise with no rejection handler,
  and a `'complete'` listener that is never released. The enclosing `try/catch`
  catches none of these, because they are promise rejections rather than throws.
  (HW-19-0028)
- **`Logger.info(\`${$r('app.string.pull_up_fail')} ...\`)` is incorrect.** A
  `Resource` interpolated into a template string yields `[object Object]`;
  resolve it with `resourceManager.getStringSync(...)` - which the same file does
  correctly for the dialog titles. (HW-19-0029)
- **Do not treat the success branch as a security boundary by itself.** The
  sample never reads `result.token`; the gate protects a client-side navigation
  only. If a server must trust the authentication, the challenge must originate
  server-side and the token must be verified there.
- **`filePre` hardcodes `mimeType: 'application/pdf'`.** It works for the one
  sample document; a general file list must derive the type from the file.

## References

- `documentation/harmonyos-references/03_system/js-apis-useriam-userauth.md` -
  `AuthParam.challenge` ("Random challenge value, which can be used to prevent
  replay attacks. It cannot exceed 32 bytes"), `WidgetParam.title` limits and the
  `uiContext` 2-in-1 restriction, `UserAuthResult.result`, `getUserAuthInstance`
  and its required `ohos.permission.ACCESS_BIOMETRIC`, `on('result')` /
  `off('result')` and the same-instance rule, and the
  `cryptoFramework.createRandom()` challenge-generation examples.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-useriam-userauth
- `documentation/harmonyos-references/03_system/js-apis-request.md` -
  `request.downloadFile`, `DownloadTask.on('complete'|'pause'|'remove')` and its
  `off` counterpart.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-request
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` -
  `accessSync`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-guides/04_system/app-permission-mgmt-overview.md` -
  permission principles and system_grant vs user_grant.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/app-permission-mgmt-overview
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/user_authentication-0000002290451165
