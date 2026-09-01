---
id: LIFE-01
title: Government-service app shell - one entry HAP over seven HARs with a dynamic-import Navigation router
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/01_practice-convenient-life-app-architecture-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1-0000001952539489
sample: huawei_industry_tree/02_convenient_life/downloads/Life_Framework_Code_V1.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.ArkWeb", "@kit.ScanKit", "@kit.VisionKit", "@kit.ImageKit", "@kit.CoreFileKit", "@kit.AssetStoreKit", "@kit.ArkTS", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [UIAbility, "windowStage.getMainWindowSync", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.on('avoidAreaChange')", "window.off('avoidAreaChange')", NavPathStack, Navigation, NavDestination, wrapBuilder, WrappedBuilder, "dynamic import()", Tabs, TabContent, TabsController, "AppStorage.setOrCreate", "AppStorage.ref", "AppStorage.setAndRef", "@StorageProp", "@Provide", "@Link", "@Prop", CardRecognition, CardType, CardRecognitionResult, "scanBarcode.startScanForResult", "generateBarcode.createBarcode", "asset.add", Web, WebviewController, "hilog.info"]
permissions: ["ohos.permission.INTERNET"]
min_api: 24
modules: [entry, common, network, home, message, mine, office, routerModule]
findings: [HW-02-0001, HW-02-0002, HW-02-0003, HW-02-0004, HW-02-0005, HW-02-0006, HW-02-0007, HW-02-0008, HW-02-0009, HW-02-0010, HW-02-0268, HW-02-0271]
status: verified-with-fixes
---

## When to use

Load this card when you are laying out a **multi-module HarmonyOS application
that ships as a single entry HAP** and needs cross-module page navigation
without a router table in `module.json5`.

The concrete product is a government-service (政务云, "government cloud") app:
a home tab with scan and identity code, a services tab, an interaction tab, and
an account tab gated behind login. The reusable part is the shell: **one `entry`
HAP, shared `common`/`network` HARs, and one HAR per bottom tab**, tied together
by a `RouterModule` singleton that maps a string key to a `WrappedBuilder` and
pulls the owning HAR in with a dynamic `import()`.

Reach for it when tabs are owned by different teams, when each feature must be
independently obfuscated (every HAR carries its own `obfuscation-rules.txt`), or
when the entry HAP must not statically depend on every page in the product.

Do **not** take this sample's login, encryption, or key-asset storage as a
pattern - all three are defective here (`HW-02-0007`, `HW-02-0008`,
`HW-02-0010`), and the document says as much at line 156: 框架代码中登录验证模块，
只是UI能力 ("the login verification module in the framework code is UI capability
only").

## Feature checklist

- Four bottom tabs - 首页 (Home), 办事 (Services), 互动 (Interaction), 我的 (Mine) -
  drawn by a custom tab bar, not by `Tabs`' own bar.
- Tapping the fourth tab with no session pushes a login page instead of
  switching tabs; swiping between tabs is disabled so the gate cannot be
  bypassed.
- Full-screen immersive layout: top and bottom avoid-area insets are read once
  at window-stage creation and kept current through an `avoidAreaChange`
  subscription.
- Every page outside the entry HAP is reached by pushing a string key onto one
  shared `NavPathStack`; the owning HAR is loaded on demand.
- Home offers barcode scanning (Scan Kit), a generated QR identity code, and
  card recognition (Vision Kit) that fills name / card type / card number.
- A generic `Web` host page for H5 content, isolated in its own `network` HAR.
- Short sensitive data goes through Asset Store Kit - **described but not
  wired**, see `HW-02-0010`.

## Architecture

Eight modules: one HAP, seven HARs. The root `build-profile.json5` declares all
eight; `entry/oh-package.json5` depends on all seven.

```
entry (HAP)                      EntryAbility + TabView, owns the single NavPathStack
├── common      (HAR)            CommonConstants, an empty mainPage placeholder
├── network     (HAR)            webPage.ets - the Web/H5 host
└── features
    ├── home    (HAR)            HomePage, Code, Credentials, ChequeSheet, RentingHouse,
    │                            ResidencePermit, SocialSecurity + Util + viewmodle
    ├── message (HAR)            MessagePage
    ├── mine    (HAR)            LoginPage, MyPage, TopPage
    ├── office  (HAR)            Office page
    └── routerModule (HAR)       RouterModule, RouterModel, RouterConstants, Logger
```

Three tree entries in the document do not match this layout - see `HW-02-0002`
(`RouterModule` vs `routerModule`, `mine/pages` vs `mine/page`, `compoents` vs
`components`) and `HW-02-0001` (`SecurityCenter.ets` was never shipped).

**The router is the piece worth copying.** It replaces the declarative
`routerMap` profile with two static maps and a naming convention:

```
BuilderNameConstants.credentials = 'home_Credentials'
                                    ^^^^ HAR name          -> what import() loads
                                        ^^^^^^^^^^^^ page key -> what pushPath() names
```

Data flow for one cross-module navigation:

1. A page calls `buildRouterModel(RouterNameConstants.ENTRY_HAP, 'home_Credentials')`.
2. `RouterModule.push` splits on `_`, takes `home`, and calls `import('home')`.
3. That resolves `features/home/Index.ets`, whose `harInit(builderName)` runs a
   `switch` and dynamically imports the one page file for that key.
4. Importing the page file executes its module-level tail:
   `if (!RouterModule.getBuilder(name)) RouterModule.registerBuilder(name, wrapBuilder(XxxBuilder))`.
5. Back in `push`, `getRouter('EntryHap_Router')?.pushPath({ name, param })`.
6. `Navigation`'s `.navDestination(this.routerMap)` looks the key up in
   `builderMap` and builds it.

The registration in step 4 is a **side effect of importing the page module** -
that is why every page file ends with the same four lines, and why the `harInit`
switch in each HAR's `Index.ets` must list every routable page.

Avoid-area insets travel the other way: `EntryAbility` writes `topRectHeight`
and `bottomRectHeight` into `AppStorage`, and every page reads them with
`@StorageProp` and converts with `getUIContext().px2vp(...)`.

## Implementation steps

1. **Create the module set** in the root `build-profile.json5`: `entry` with
   `targets/applyToProducts`, and one bare `{ name, srcPath }` entry per HAR.
   Set `compatibleSdkVersion` and `targetSdkVersion` to `6.1.1(24)` and enable
   `strictMode.useNormalizedOHMUrl`.
2. **Build `routerModule` first** - everything else depends on it. It has no
   dependencies of its own: two static `Map`s, `registerBuilder`/`getBuilder`,
   `createRouter`/`getRouter`, and `push`/`pop`/`clear`/`popToName`.
3. **Fix the naming convention before writing any page**: `<harName>_<PageName>`.
   The HAR name must equal the dependency key in `oh-package.json5`, because
   `push` feeds it straight to `import()`.
4. **In `EntryAbility.onWindowStageCreate`**, load `pages/Index`, take the main
   window, call `setWindowLayoutFullScreen(true)`, read both avoid areas into
   `AppStorage`, and subscribe to `avoidAreaChange`. **Keep the window handle
   and the callback in fields** so `onWindowStageDestroy` can call
   `off('avoidAreaChange', cb)` - the sample never does (`HW-02-0006`).
5. **Write the entry page** as a `Navigation(this.pageStack)` wrapping a `Tabs`
   with `barHeight(0)` and a hand-rolled tab bar underneath. Register the stack
   in `aboutToAppear` with
   `RouterModule.createRouter(RouterNameConstants.ENTRY_HAP, this.pageStack)`,
   and pass `this.routerMap` to `.navDestination()`.
6. **Gate the account tab** inside the tab item's `onClick`: read
   `AppStorage.ref<LoginInfo>('login')?.get()`; if it is missing or
   `isLogin === false`, call `buildRouterModel(..., BuilderNameConstants.loginPage)`
   instead of assigning `selectedIndex`. Mirror the guard in `Tabs.onChange` and
   set `.scrollable(false)`.
7. **Give each feature HAR an `Index.ets`** that statically re-exports the
   tab-level component and exposes `harInit(builderName)` with a `switch` of
   dynamic `import()` calls for the pushed pages.
8. **End every routable page file** with the register-if-absent block and
   `wrapBuilder`. Wrap the page body in `NavDestination()` and capture the stack
   in `.onReady((ctx) => this.pathStack = ctx.pathStack)`.
9. **For card recognition**, mount `CardRecognition` conditionally
   (`if (this.isClicked)`) with `.opacity(0)`, and use `onResult`, not `callback`
   (`HW-02-0004`). Guard `cardInfo?.back?.originalImageUri` before opening it and
   close the descriptor in `finally` (`HW-02-0005`, `HW-02-0003`).
10. **For secrets, do not follow the sample.** Take the key from HUKS or Asset
    Store Kit and generate a fresh random IV per call (`HW-02-0007`), and never
    leave the rejection handler empty (`HW-02-0008`).

## Verified snippets

All snippets are from `Life_Framework_Code_V1.zip`. Corrected forms are marked.

**The router core - `Life_Framework_Code_V1.zip#features/routerModule/src/main/ets/utils/RouterModule.ets`** (as shipped)

```typescript
function callHarInit(ns: ESObject, router: RouterModel) {
  ns.harInit(router.builderName);
}

export class RouterModule {
  static builderMap: Map<string, WrappedBuilder<[object]>> = new Map<string, WrappedBuilder<[object]>>();
  static routerMap: Map<string, NavPathStack> = new Map<string, NavPathStack>();

  public static registerBuilder(builderName: string, builder: WrappedBuilder<[object]>): void {
    RouterModule.builderMap.set(builderName, builder);
  }

  public static getBuilder(builderName: string): WrappedBuilder<[object]> | undefined {
    const builder = RouterModule.builderMap.get(builderName);
    if (!builder) {
      Logger.info('not found builder ' + builderName);
    }
    return builder;
  }

  public static createRouter(routerName: string, router: NavPathStack): void {
    RouterModule.routerMap.set(routerName, router);
  }

  public static async push(router: RouterModel): Promise<void> {
    const harName = router.builderName.split('_')[0];
    import(harName).then((ns: ESObject) => {
      callHarInit(ns, router);
      RouterModule.getRouter(router.routerName)?.pushPath({ name: router.builderName, param: router.param });
    }).catch((err: BusinessError) => {
      hilog.info(0x0000, 'ccTest', err.message);
    });
  }
}
```

**`split('_')[0]` is the whole module-resolution strategy.** Because the key is
`home_Credentials`, `import('home')` resolves through `entry/oh-package.json5`'s
`"home": "file:../features/home"` to that HAR's `Index.ets`. Rename a HAR
without renaming its keys and navigation fails at runtime, not at build time -
the `.catch` only logs.

`push` is declared `async` but returns before the import settles, so `pushPath`
happens after the caller has already continued. Nothing here awaits it, but you
cannot rely on the page being on the stack when `buildRouterModel` returns.

**Per-HAR lazy loading - `Life_Framework_Code_V1.zip#features/home/Index.ets`** (as shipped)

```typescript
import { BuilderNameConstants } from 'routermodule';

export { HomePage } from './src/main/ets/pages/HomePage';

export function harInit(builderName: string): void {
  switch (builderName) {
    case BuilderNameConstants.chequeSheet:
      import('./src/main/ets/pages/ChequeSheetPage');
      break;
    case BuilderNameConstants.credentials:
      import('./src/main/ets/pages/CredentialsPage');
      break;
    case BuilderNameConstants.code:
      import('./src/main/ets/pages/CodePage');
      break;
    default:
      break;
  }
}
```

The tab-level component (`HomePage`) is a **static** export - the entry HAP
renders it immediately. Everything reachable only by navigation sits behind
`harInit`, so its code is not pulled into the first frame.

**The registration tail every routable page repeats - `Life_Framework_Code_V1.zip#features/home/src/main/ets/pages/CodePage.ets:52`**

```typescript
const builderName = BuilderNameConstants.code;
if (!RouterModule.getBuilder(builderName)) {
  const builder: WrappedBuilder<[object]> = wrapBuilder(CodeBuilder);
  RouterModule.registerBuilder(builderName, builder);
}
```

The `if` guard matters: `harInit` may be called again for the same key on a
second navigation. Re-importing a loaded module is a no-op, but the guard makes
the block explicitly idempotent.

**Entry shell - `Life_Framework_Code_V1.zip#entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
@Entry
@Component
export struct TabView {
  @StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
  @Provide selectedIndex: number = 0;
  private pageStack: NavPathStack = new NavPathStack();
  private controller: TabsController = new TabsController();

  aboutToAppear(): void {
    if (!this.pageStack) {
      this.pageStack = new NavPathStack();
    }
    RouterModule.createRouter(RouterNameConstants.ENTRY_HAP, this.pageStack);
  }

  @Builder
  routerMap(builderName: string, param: object) {
    RouterModule.getBuilder(builderName)?.builder(param);
  };

  build() {
    Column() {
      Navigation(this.pageStack) {
        Tabs({ index: this.selectedIndex, barPosition: BarPosition.End, controller: this.controller }) {
          TabContent() { Column() { HomePage({ homePageStack: this.pageStack }); }; };
          TabContent() { Column() { OfficePage(); }; };
          TabContent() { MessagePage(); };
          TabContent() { MyPage(); };
        }
        .vertical(false)
        .scrollable(false)
        .layoutWeight(1)
        .barHeight($r('app.integer.custom_tab_common_size_0'))   // 0 - the built-in bar is hidden
        .onChange((index: number) => {
          if (index !== 3) { this.selectedIndex = index; }
        });

        CustomTabBar({ selectedIndex: $selectedIndex })
          .padding({ bottom: this.getUIContext().px2vp(this.bottomRectHeight) });
      }
      .hideTitleBar(true)
      .navDestination(this.routerMap);
    }
    .width('100%')
    .height('100%');
  }
}
```

**One `NavPathStack` for the whole application.** It is created here, handed to
`RouterModule` under a single name, and also passed into `HomePage` as a `@Prop`
because that page hosts its own nested `Navigation`. Every HAR pushes onto this
one stack, which is what makes the string-key router work across module
boundaries.

`.barHeight(0)` plus a sibling `CustomTabBar` is the idiom for a fully custom
tab bar: `Tabs` still owns content switching and the index, but draws no chrome.
`.scrollable(false)` stops a swipe from bypassing the login gate.

**The login gate - same file, `TabItem.onClick`** (as shipped)

```typescript
.onClick(() => {
  if (this.tabBarIndex === 3) {
    let loginInfo: LoginInfo | null | undefined = AppStorage.ref<LoginInfo>('login')?.get();
    let isLogin: boolean = loginInfo ? loginInfo.isLogin : false;
    if (!isLogin) {
      buildRouterModel(RouterNameConstants.ENTRY_HAP, BuilderNameConstants.loginPage);
    } else {
      this.selectedIndex = this.tabBarIndex;
    }
  } else {
    this.selectedIndex = this.tabBarIndex;
  }
});
```

`AppStorage.ref<T>(key)` returns `undefined` when the key was never created, so
`?.get()` plus the ternary is the complete "no session yet" path. The matching
`if (index !== 3)` in `Tabs.onChange` stops `Tabs` committing the index behind
the gate's back.

**Immersive insets - `Life_Framework_Code_V1.zip#entry/src/main/ets/entryability/EntryAbility.ets:48`** (corrected, see `HW-02-0006`)

```typescript
private windowClass?: window.Window;
private avoidAreaCallback = (data: window.AvoidAreaOptions) => {   // FIX: named, so it can be released
  if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
    AppStorage.setOrCreate('topRectHeight', data.area.topRect.height);
  } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
    AppStorage.setOrCreate('bottomRectHeight', data.area.bottomRect.height);
  }
};

onWindowStageCreate(windowStage: window.WindowStage): void {
  windowStage.loadContent('pages/Index', (err) => { /* ... */ });
  this.windowClass = windowStage.getMainWindowSync();

  this.windowClass.setWindowLayoutFullScreen(true)
    .catch((err: BusinessError) => {
      hilog.error(DOMAIN, 'EntryAbility', 'fullscreen failed: %{public}s', JSON.stringify(err));
    });

  let avoidArea = this.windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR);
  AppStorage.setOrCreate('bottomRectHeight', avoidArea.bottomRect.height);
  avoidArea = this.windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM);
  AppStorage.setOrCreate('topRectHeight', avoidArea.topRect.height);

  this.windowClass.on('avoidAreaChange', this.avoidAreaCallback);
}

onWindowStageDestroy(): void {
  this.windowClass?.off('avoidAreaChange', this.avoidAreaCallback);  // FIX: the sample has no off() at all
  this.windowClass = undefined;
}
```

**Read once, then subscribe.** `getWindowAvoidArea` gives the value at startup;
the subscription keeps it correct across rotation, folding, and keyboard show.
Both are needed - the callback does not fire until something changes. The heights
are in **px**, which is why every consumer wraps them in
`getUIContext().px2vp(...)`.

**Card recognition - `Life_Framework_Code_V1.zip#features/home/src/main/ets/pages/CredentialsPage.ets:170`** (corrected, see `HW-02-0005`)

```typescript
if (this.isClicked) {
  CardRecognition({
    supportType: CardType.CARD_ID,
    onResult: (async (params: CardRecognitionResult) => {        // 'onResult', not 'callback' (HW-02-0004)
      if (params.code) {
        this.isClicked = false;
      }
      const backUri = params.cardInfo?.back?.originalImageUri;    // FIX: ?. on back as well
      if (backUri) {
        const file = fs.openSync(backUri, fs.OpenMode.READ_ONLY);
        try {
          const imageSource = image.createImageSource(file.fd);
          this.chooseImage = await imageSource.createPixelMap();
        } finally {
          fs.closeSync(file);                                     // the shipped code does close it
        }
      }
      if (params.cardInfo?.front !== undefined && params.cardType) {  // FIX: hoisted out of the image branch
        this.cardDataSource.push(params.cardInfo.front);
        this.cardType = cardTypeString(params.cardType);
        this.name = this.cardDataSource[0].name;
        this.idNumber = this.cardDataSource[0].idNumber;
      }
    })
  })
    .opacity(0);
}
```

**`CardRecognition` is a full-screen control, mounted invisibly.** The
`if (this.isClicked)` wrapper is what launches it - the component takes over the
screen when it enters the tree - and `.opacity(0)` keeps it from drawing over
the form underneath while mounted. Setting `isClicked` back to `false` in the
result handler unmounts it.

In the shipped code the field extraction sits **inside** the back-image block, so
a scan that returns only the front fills nothing in; hoisting it out is part of
the `HW-02-0005` fix.

**Identity QR code - `Life_Framework_Code_V1.zip#features/home/src/main/ets/Util/UtilTools.ets:31`** (as shipped)

```typescript
async creatQR(): Promise<image.PixelMap | undefined> {
  let px: image.PixelMap | undefined = undefined;
  let content: string = 'huawei';
  let options: generateBarcode.CreateOptions = {
    scanType: scanCore.ScanType.QR_CODE,
    height: 400,
    width: 400
  };
  await generateBarcode.createBarcode(content, options).then((pixelMap: image.PixelMap) => {
    px = pixelMap;
    return pixelMap;
  }).catch((error: BusinessError) => {
    hilog.error(0x0001, '[generateBarcode]',
      `Failed to get PixelMap by promise with options. Code: ${error.code}, message: ${error.message}`);
  });
  return px;
}
```

The `PixelMap` is generated in `HomePage.aboutToAppear`, parked in a
`GlobalContext` singleton `Map`, and read back by `CodePage.aboutToAppear`. That
is how the sample passes a non-serializable object between two HAR pages without
putting it on the navigation stack. It is never released, so in production add a
`release()` when the code page is torn down.

**Barcode scanning - `Life_Framework_Code_V1.zip#features/home/src/main/ets/pages/HomePage.ets:127`** (as shipped)

```typescript
let options: scanBarcode.ScanOptions = {
  scanTypes: [scanCore.ScanType.ALL],
  enableMultiMode: true,
  enableAlbum: true
};
let context = this.getUIContext().getHostContext();
scanBarcode.startScanForResult(context, options).then((result: scanBarcode.ScanResult) => {
  hilog.info(0x0001, '[Scan CPSample]', `result is ${JSON.stringify(result)}`);
}).catch((error: BusinessError) => {
  hilog.error(0x0001, '[Scan CPSample]', `Code:${error.code}, message: ${error.message}`);
});
```

`startScanForResult` launches the **system** scan UI, which is why no camera
permission appears in any `module.json5` in this project.

**H5 host - `Life_Framework_Code_V1.zip#common/network/src/main/ets/components/mainpage/webPage.ets`** (as shipped)

```typescript
Web({ controller: this.webController, src: this.url })
  .fileAccess(false)
  .geolocationAccess(false)
  .javaScriptAccess(true)
  .domStorageAccess(true)
  .onControllerAttached(() => {
    this.webController.loadUrl(this.url);
  });
```

`fileAccess(false)` and `geolocationAccess(false)` are the two switches worth
keeping for a government-service H5 container: no sandbox file reads from page
script, no silent location. `onControllerAttached` is the earliest point at
which `loadUrl` is legal on a `WebviewController`.

## Permissions & config

Exactly one permission is declared in the whole project, and it is declared in
the `home` HAR rather than in the entry HAP.

`Life_Framework_Code_V1.zip#features/home/src/main/module.json5`

```json5
{
  "module": {
    "requestPermissions": [
      { "name": "ohos.permission.INTERNET" }
    ],
    "name": "home",
    "type": "har",
    "deviceTypes": ["default", "tablet", "2in1"]
  }
}
```

This is valid: "When a HAP references a HAR, the system automatically combines
their permission configurations during compilation and build. Therefore, you do
not need to repeatedly request the same permission in the HAP and HAR."
(`documentation/harmonyos-guides/01_getting-started/har-package.md`,
Constraints). `ohos.permission.INTERNET` is system-granted, so there is no
runtime request flow.

`Life_Framework_Code_V1.zip#entry/src/main/module.json5`

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
    "abilities": [{
      "name": "EntryAbility",
      "srcEntry": "./ets/entryability/EntryAbility.ets",
      "exported": true,
      "skills": [{ "entities": ["entity.system.home"], "actions": ["action.system.home"] }]
    }]
  }
}
```

`main_pages.json` lists exactly one page - `pages/Index`. Every other page is a
`NavDestination` reached through the router. A HAR cannot declare `pages` at
all, so this is the only workable arrangement for a HAR-based feature split.

Root `build-profile.json5`:

```json5
"compatibleSdkVersion": "6.1.1(24)",
"targetSdkVersion": "6.1.1(24)",
"buildOption": { "strictMode": { "useNormalizedOHMUrl": true } }
```

`useNormalizedOHMUrl` is what lets the dynamic `import(harName)` in
`RouterModule.push` resolve a HAR by its dependency name.

The project also depends on `@hw-agconnect/petal-aegis` (root
`oh-package.json5`) for `AegAesCbc` - an AppGallery Connect package, not a
HarmonyOS kit.

## Constraints

- DevEco Studio 6.1.1 Release, API Version 24 Release (document lines 162-163;
  matches `compatibleSdkVersion` in the zip).
- Phone-first: the document states at line 16 应用只部署在手机端 ("the application
  is deployed only on mobile phones"); the HAP declares
  `deviceTypes: ["phone", "tablet", "2in1"]`.
- A HAR cannot declare the `pages` tag, cannot reference `AppScope` resources,
  and supports neither cyclic nor transitive dependency
  (`harmonyos-guides/01_getting-started/har-package.md`, Constraints). The first
  of those is why the `<har>_<Page>` key convention exists.
- The router resolves HARs by **name, at runtime**. A typo in a
  `BuilderNameConstants` value fails silently - `push`'s `.catch` only logs.
- Atomic-service reuse adds hard limits the document calls out at lines 58-59: a
  single package must be under 2 MB, and any HSP shared between the app and the
  atomic service must be written against the atomic-service high-level API
  subset.
- This is framework code only, not a complete application (document line 154:
  本篇代码非应用的全量代码, "this code is not the full code of the application").
  Login accepts any password for a valid-looking phone number.
- The `common` HAR's `mainPage.ets` is an empty component, and
  `ResidencePermitPage.ets` declares a `WebviewController` it never uses.

## Pitfalls

- **`HW-02-0004` - the document's `CardRecognition` snippet is not the shipped
  code.** It writes `callback: (async (params: CallbackParam) => ...)`; the
  sample writes `onResult: (async (params: CardRecognitionResult) => ...)` and
  imports `CardRecognitionResult` from `@kit.VisionKit`. `CallbackParam` appears
  nowhere in the project. Use `onResult`.
- **`HW-02-0003` - the document's snippet leaks a file descriptor.** It calls
  `fs.openSync` and never closes it; the zip wraps the body in
  `try { ... } finally { fs.closeSync(file); }`. Copy the zip, not the document.
- **`HW-02-0005` - `params.cardInfo?.back.originalImageUri` guards the wrong
  level.** `?.` short-circuits only `cardInfo`; when `back` is undefined the
  property access throws, and the statement sits outside the `try`. Write
  `params.cardInfo?.back?.originalImageUri` and check it before opening.
- **`HW-02-0006` - `on('avoidAreaChange')` has no `off()`.** The reference
  documents `off(type: 'avoidAreaChange', callback?)` as the counterpart; call
  it from `onWindowStageDestroy` and keep the callback in a field so it can be
  passed back.
- **`HW-02-0007` - the AES key and IV are string literals in `LoginPage.ets`.**
  The sample's own comment states the rule it breaks: 不能硬编码在代码中 ("must
  not be hardcoded in the code"). Take the key from HUKS or Asset Store Kit and
  generate the IV per call.
- **`HW-02-0008` - the encryption rejection handler is empty,** and
  `RouterModule.clear` runs regardless, so a failed login dismisses the page
  without writing any session. Log, notify, and dismiss only on success.
- **`HW-02-0009` - `hilog.info(DOMAIN, '%{public}s', JSON.stringify(err))` puts
  the format string in the tag slot.** The signature is
  `info(domain, tag, format, ...args)`. The mirror-image mistake is at
  `EntryAbility.ets:27-28`, where two arguments are supplied for one placeholder
  and the second is silently dropped.
- **`HW-02-0010` - `AssetStore.ets` is imported by nothing** and runs
  `asset.add` at module top level with the literals `demo_pwd` / `demo_alias`.
  The documented on-device secure storage scheme is not exercised by the running
  app; the account actually lands in volatile `AppStorage`.
- **`HW-02-0001` / `HW-02-0002` - the documented tree is not the shipped tree.**
  `SecurityCenter.ets` does not exist, and three directory names differ
  (`RouterModule`/`routerModule`, `pages`/`page`, `compoents`/`components`).
  Read `build-profile.json5` and `oh-package.json5`, not the document's tree.
- **Do not forget the registration tail on a new page.** A page that omits
  `RouterModule.registerBuilder(...)` imports fine, pushes fine, and then renders
  nothing - `getBuilder` returns `undefined` and `routerMap` silently builds an
  empty subtree.
- **Do not add a page to a HAR without adding it to that HAR's `harInit`
  switch.** The dynamic import never runs, so the builder never registers.
- **`AppStorage.setAndRef` is not a setter.** The reference calls it a
  get-or-create - its example notes `AppStorage.setAndRef('PropA', 50)` leaves an
  existing `PropA` at 47. `LoginPage` gets away with it only because it mutates
  the object it already holds a reference to. Use `AppStorage.setOrCreate` when
  you intend to overwrite.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `on('avoidAreaChange')` / `off('avoidAreaChange')`, `getWindowAvoidArea`, `setWindowLayoutFullScreen`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/02_application-framework/ts-state-management.md` - `AppStorage.ref`, `AppStorage.setAndRef` (get-or-create, not a setter), `AppStorage.setOrCreate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-state-management
- `documentation/harmonyos-references/03_system/js-apis-hilog.md` - `info(domain, tag, format, ...args)` and the placeholder/argument rule
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
- `documentation/harmonyos-references/03_system/js-apis-asset.md` - `asset.add`, `asset.Tag`, `asset.Accessibility.DEVICE_FIRST_UNLOCKED`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-asset
- `documentation/harmonyos-guides/01_getting-started/har-package.md` - HAR constraints: no `pages` tag, no `AppScope` resources, permission merging into the referencing HAP
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/har-package
- `documentation/harmonyos-guides/01_getting-started/module-configuration-file.md` - `module.json5` tags, `requestPermissions`, `routerMap`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/module-configuration-file
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - declaring permissions in the configuration file
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `documentation/harmonyos-references/07_ai/vision-component.md` - the `CardRecognition` control's entry in the ArkTS component list
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/vision-component
