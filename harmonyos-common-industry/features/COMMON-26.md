---
id: COMMON-26
title: Splash screen ad integration - load an Ads Kit splash ad with a timeout, then route into the app
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/26_splash_page_ad_access.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/splash_page_ad_access-0000002322822425
sample: huawei_industry_tree/19_common_technical_solutions/downloads/OpenScreenAds.zip
kits: ["@kit.AdsKit", "@kit.ArkUI", "@kit.AbilityKit", "@kit.ArkData", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["advertising.AdLoader", "AdLoader.loadAd", "advertising.AdRequestParams", "advertising.AdOptions", "advertising.AdLoadListener", "advertising.Advertisement", AdComponent, "AdInteractionListener.onStatusChanged", "window.getLastWindow", "Window.setWindowLayoutFullScreen", "Window.setPreferredOrientation", "window.Orientation", "UIContext.getRouter", "Router.replaceUrl", "router.RouterMode", "preferences.getPreferences", "Preferences.putSync", "Preferences.flushSync", "Preferences.getSync", "ApplicationContext.restartApp", RelativeContainer, TransitionEffect]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0081, HW-19-0082, HW-19-0083, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when the application should **show an advertisement on its splash
screen before the home page appears** - request the ad on launch, display it
full-screen or half-screen, let the user tap it or skip it, and enter the app
either way.

The two things that make this different from "show an image at launch" are the
**timeout** (the app must not be held hostage by a slow ad request) and the
**status callback** (the ad component, not the app, draws the skip control and
reports when the ad closes).

## Feature checklist

The application must:

- Put the splash page first and route to the home page from it.
- Set the window full-screen and lock the orientation while the splash is up, and
  restore both when it leaves.
- Start an ad request in `aboutToAppear` with an ad slot ID, ad type and count.
- Arm a timeout at the same moment and route to the home page if the ad has not
  arrived in time.
- Cancel that timeout once the outcome is known - on **every** path
  (HW-19-0083).
- Render the returned ad through `AdComponent`, choosing the full-screen or
  half-screen layout from `ad.isFullScreen`.
- Route to the home page when the interaction listener reports `AD_CLOSED`.
- Handle the window-lookup error rather than assuming a window (HW-19-0082).
- Be honest about the tracking permission (HW-19-0081).

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup; publishes the context into `AppStorage`; initialises `PreferencesUtil` |
| `pages/SplashAdPage.ets` | the splash: window mode, ad request, timeout, slogan artwork; hosts `AdView` |
| `view/AdView.ets` | picks `splashFullScreen()` or `splashHalfScreen()` from `ad.isFullScreen` and mounts `AdComponent` |
| `enums/AdStatus.ets` | the status strings compared in `onStatusChanged` |
| `router/RouterManager.ets` | one static `routeToHome(uiContext)` |
| `utils/PreferencesUtil.ets` | a static Preferences facade with JSON auto-encoding |
| `components/CustomButton.ets` | the two demo buttons on the home page |
| `pages/Index.ets` | the home page - and the demo switch that restarts the app in the other ad mode |

**Launch flow.**

1. `SplashAdPage.aboutToAppear` obtains the top window, sets
   `setWindowLayoutFullScreen(true)` and
   `setPreferredOrientation(window.Orientation.PORTRAIT)`.
2. It reads the demo's `isFullScreen` preference and picks one of the two test ad
   slot IDs.
3. `loadAd(adId)` builds `AdRequestParams` (`adType: 1` = splash, `adCount: 1`,
   `oaid: ''`) and `AdOptions`, creates an `advertising.AdLoader(context)`, arms
   the timeout, and calls `adLoader.loadAd(params, options, listener)`.
4. `onAdLoadSuccess` clears the timeout, drops the result if the timeout already
   fired, and assigns `this.ad` - which makes `AdView` render.
5. `AdComponent`'s `onStatusChanged` reports `AD_OPEN`, `AD_CLICKED` and
   `AD_CLOSED`; the last routes home.
6. `aboutToDisappear` restores the window to non-full-screen and
   `Orientation.UNSPECIFIED`.

**The demo switch.** `Index.restartApp(isFullScreen)` writes the preference and
calls `applicationContext.restartApp(want)` - the ad type is fixed at request
time, so the sample restarts the process to demonstrate the other layout. That is
demo scaffolding, not part of the ad integration.

**Nothing draws a skip button.** The document says 点击"跳过"进入应用主页面 ("tap
Skip to enter the app home page"); the skip control belongs to `AdComponent`, and
the application only reacts to the resulting `AD_CLOSED` status.

## Implementation steps

1. **Set up AGC.** The document notes: 实现广告加载功能需在AGC上申请调试证书
   ("a debug certificate must be applied for in AGC to implement ad loading").
   Without it the request cannot succeed even with correct code.
2. **Make the splash the launch page** and give it a `RouterManager` that
   `replaceUrl`s to the home page, so the splash cannot be returned to.
3. **Take over the window in `aboutToAppear`** - full-screen plus portrait - and
   check the `getLastWindow` error before touching the window (HW-19-0082).
4. **Build the request.** `adId` (test IDs during development, real slot IDs
   before release), `adType: 1`, `adCount: 1`, and `oaid` - empty for
   non-personalised, or a real OAID if you take the tracking-consent path
   (HW-19-0081).
5. **Build `AdOptions` deliberately.** `allowMobileTraffic`,
   `tagForChildProtection`, `tagForUnderAgeOfPromise` and
   `adContentClassification` are compliance settings, not styling.
6. **Arm the timeout before the request**, and clear it in `onAdLoadSuccess`,
   `onAdLoadFailure` and `aboutToDisappear` (HW-19-0083).
7. **Guard against a late success** with the `isTimeOut` flag, so an ad arriving
   after the user has been routed home is discarded.
8. **Render through `AdComponent`**, branching on `ad.isFullScreen`, and route
   home on `AD_CLOSED`.
9. **Restore the window in `aboutToDisappear`.**

## Verified snippets

All snippets below come from the sample project, not from the document.

### Window handling and the ad request kick-off

`OpenScreenAds.zip#OpenScreenAds/entry/src/main/ets/pages/SplashAdPage.ets`

```ts
// 'testd7c5cewoj6'、'testq6zq98hecj'为测试专用的广告位ID，App正式发布时需要改为正式的广告位ID
const VIDEO_TEST_AD_ID = 'testd7c5cewoj6';
const IMAGE_TEST_AD_ID = 'testq6zq98hecj';

aboutToAppear() {
  window.getLastWindow(this.context, (err: BusinessError, win: window.Window) => {
    // FIX (HW-19-0082): if (err.code) { hilog.error(...); return; }
    // 开启全屏模式沉浸页面
    win.setWindowLayoutFullScreen(true);
    // 设置屏幕方向为竖屏
    win.setPreferredOrientation(window.Orientation.PORTRAIT);
  });
  // 调用loadAd加载广告
  let isFullScreen = PreferencesUtil.getSync('isFullScreen', false) as boolean;
  let adId = isFullScreen ? VIDEO_TEST_AD_ID : IMAGE_TEST_AD_ID;
  this.loadAd(adId);
}

aboutToDisappear(): void {
  window.getLastWindow(this.context, (err: BusinessError, win: window.Window) => {
    // 关闭全屏模式，开发者可根据实际情况修改
    win.setWindowLayoutFullScreen(false);
    // 设置屏幕方向为默认值，开发者可根据实际情况修改
    win.setPreferredOrientation(window.Orientation.UNSPECIFIED);
  });
  // FIX (HW-19-0083): clearTimeout(this.timeOutIndex);
}
```

### The ad request

`OpenScreenAds.zip#OpenScreenAds/entry/src/main/ets/pages/SplashAdPage.ets`

```ts
// 请求广告
private async loadAd(adId: string): Promise<void> {
  // 广告请求参数
  const adRequestParams: advertising.AdRequestParams = {
    // 广告位ID
    adId: adId,
    // 开屏广告类型
    adType: 1,
    // 请求的广告数量
    adCount: 1,
    // 开放匿名设备标识符
    oaid: ''                       // FIX (HW-19-0081)
  };
  // 广告配置参数
  const adOptions: advertising.AdOptions = {
    // 是否允许流量下载 0：不允许，1：允许，不设置以广告主设置为准
    allowMobileTraffic: 1,
    // 是否希望根据 COPPA 的规定将您的内容视为面向儿童的内容: -1: 默认值，不确定 0: 不希望 1: 希望
    tagForChildProtection: -1,
    // 是否希望按适合未达到法定承诺年龄的欧洲经济区 (EEA) 用户的方式处理该广告请求
    tagForUnderAgeOfPromise: -1,
    // 设置广告内容分级上限 W: 3+,所有受众 PI: 7+,家长指导 J: 12+,青少年 A: 16+/18+,成人受众
    adContentClassification: 'A'
  };
  // 广告请求回调监听
  const adLoadListener: advertising.AdLoadListener = {
    // 广告请求失败回调
    onAdLoadFailure: (errorCode: number, errorMsg: string) => {
      hilog.error(0x0000, 'testTag', `Failed to load ad. Code is ${errorCode}, message is ${errorMsg}`);
      // FIX (HW-19-0083): clearTimeout + route home explicitly
    },
    // 广告请求成功回调
    onAdLoadSuccess: (ads: Array<advertising.Advertisement>) => {
      clearTimeout(this.timeOutIndex);
      if (this.isTimeOut) {
        return;
      }
      hilog.info(0x0000, 'testTag', 'Succeeded in loading ad');
      this.ad = ads[0];
    }
  };
  // 创建AdLoader广告对象
  const adLoader: advertising.AdLoader = new advertising.AdLoader(this.context);
  this.timeOutHandler();
  // 调用广告请求接口
  adLoader.loadAd(adRequestParams, adOptions, adLoadListener);
}

private timeOutHandler(): void {
  this.isTimeOut = false;
  // 超时处理
  this.timeOutIndex = setTimeout(() => {
    this.isTimeOut = true;
    RouterManager.routeToHome(this.uiContext);
    hilog.error(0x0000, 'testTag', 'Load ad time out');
  }, this.timeOutDuration);
}
```

### Rendering the ad

`OpenScreenAds.zip#OpenScreenAds/entry/src/main/ets/view/AdView.ets`

```ts
build() {
  if (this.ad) {
    if (this.ad.isFullScreen === true) {
      this.splashFullScreen();
    } else {
      this.splashHalfScreen();
    }
  }
}

@Builder
private splashHalfScreen() {
  AdComponent({
    ads: [this.ad!],
    displayOptions: this.adDisplayOptions,
    interactionListener: {
      onStatusChanged: (status: string) => {
        switch (status) {
          case AdStatus.AD_OPEN:
            hilog.info(0x0000, 'testTag', 'Status is onAdOpen');
            break;
          case AdStatus.AD_CLICKED:
            hilog.info(0x0000, 'testTag', 'Status is onAdClick');
            break;
          case AdStatus.AD_CLOSED:
            hilog.info(0x0000, 'testTag', 'Status is onAdClose');
            RouterManager.routeToHome(this.uiContext);
            break;
        }
      }
    }
  })
    .zIndex(1)
    .width('100%')
    .height('90%')// 自定义组件动画
    .transition(TransitionEffect.OPACITY.animation({ duration: 1000, curve: Curve.Friction, delay: 500 }))
    .alignRules({ top: { anchor: '__container__', align: VerticalAlign.Top } });
}
```

The full-screen builder is identical except for `height('100%')` and no
transition or alignment rules.

### Routing into the app

`OpenScreenAds.zip#OpenScreenAds/entry/src/main/ets/router/RouterManager.ets`

```ts
export class RouterManager {
  /**
   * 跳转至主界面
   */
  static routeToHome(uiContext: UIContext): void {
    // 开发者可根据项目实际情况修改超时之后要跳转的目标页面
    uiContext.getRouter().replaceUrl({ url: 'pages/Index' }, router.RouterMode.Single);
  }
}
```

`replaceUrl` rather than `pushUrl` is the right choice - the splash must not
remain on the back stack.

### The preferences facade

`OpenScreenAds.zip#OpenScreenAds/entry/src/main/ets/utils/PreferencesUtil.ets`

```ts
export class PreferencesUtil {
  private static instance: PreferencesUtil = new PreferencesUtil();
  private preferences: preferences.Preferences | null = null;

  // 初始化首选项（建议在应用启动时调用）
  public static async initPreferences(context: Context, name: string = 'myAppData') {
    try {
      PreferencesUtil.instance.preferences = await preferences.getPreferences(context, name);
    } catch (err) {
      hilog.error(0x0000, 'testTag', `Failed to init preferences. Code:${err.code},message:${err.message}`);
    }
  }

  // 同步读取方法
  public static getSync<T>(key: string, defaultValue: ValueType): T {
    if (!PreferencesUtil.instance.preferences) {
      hilog.error(0x0000, 'testTag', 'Preferences not initialized!');
      return defaultValue as T;
    }
    try {
      const value = PreferencesUtil.instance.preferences.getSync(key, defaultValue);
      // 自动解析JSON字符串
      if (typeof value === 'string' && PreferencesUtil.isJsonString(value)) {
        return JSON.parse(value);
      }
      return value as T;
    } catch (err) {
      hilog.error(0x0000, 'testTag', `Get failed. Code:${err.code},message:${err.message}`);
      return defaultValue as T;
    }
  }
}
```

## Permissions & config

**The sample declares no permissions.**
`OpenScreenAds.zip#OpenScreenAds/entry/src/main/module.json5` has no
`requestPermissions` block at all - only the `EntryAbility` with the home skill,
an `EntryBackupAbility` backup extension and
`"deviceTypes": ["phone","tablet","2in1"]`.

That contradicts the document's 权限说明 section, which requires
`ohos.permission.APP_TRACKING_CONSENT` (HW-19-0081). The permission exists to let
the application read the open anonymous device identifier (OAID) for
personalised ads; since the sample passes `oaid: ''` it takes the
non-personalised path and needs nothing. If you want the personalised path:

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.APP_TRACKING_CONSENT",
    "reason": "$string:reason_for_tracking",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
]
```

plus a runtime `requestPermissionsFromUser` call and a real OAID in
`AdRequestParams`.

**AGC configuration is a hard prerequisite** - the document states
实现广告加载功能需在AGC上申请调试证书 ("a debug certificate must be applied for in
AGC to implement the ad loading function"). The two ad IDs in the source are test
slots and the comment says so: 'App正式发布时需要改为正式的广告位ID' ("must be
changed to the formal ad slot ID when the app is officially released").

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **An AGC debug certificate is required** before any ad can load, so the sample
  shows only the slogan artwork until that is set up.
- **The ad slot IDs must be replaced before release** - the two in the source are
  test IDs.
- **`adType: 1` is the splash ad type**, and `adCount: 1` is what a splash
  placement expects.
- **The `AdOptions` compliance fields are not decoration.**
  `tagForChildProtection` and `tagForUnderAgeOfPromise` carry COPPA and EEA
  obligations, and `adContentClassification: 'A'` is the 16+/18+ ceiling - review
  these per market rather than copying.
- **The skip control belongs to `AdComponent`.** The application only receives
  `AD_CLOSED`; it cannot restyle or reposition the skip affordance.
- **`ad.isFullScreen` decides the layout**, and it comes from the served ad, not
  from the request - both layouts must exist.
- **The window changes are global.** `setWindowLayoutFullScreen` and
  `setPreferredOrientation` affect the whole window, which is why
  `aboutToDisappear` has to restore them.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **The document requires `ohos.permission.APP_TRACKING_CONSENT`, but the sample
  declares nothing and sends `oaid: ''`.** The permission section describes the
  personalised-ad path; the code implements the non-personalised one. Pick one
  and make both halves agree. (HW-19-0081)
- **Both `getLastWindow` callbacks ignore the error and use the window, which is
  incorrect.** The reference example reads `err.code` and returns first; without
  that the failure path calls `setWindowLayoutFullScreen` on a non-window, and
  in `aboutToDisappear` a silent failure leaves the app locked in portrait.
  (HW-19-0082)
- **The timeout is cleared only on success, which is incorrect.**
  `onAdLoadFailure` and `aboutToDisappear` leave it armed, so a stale timer can
  `replaceUrl` after the page is gone, and a failed request is reported as a
  timeout. (HW-19-0083)
- **`onAdLoadFailure` does nothing but log.** The user waits out the full
  timeout for an error that was already known.
- **`this.ad = ads[0]` takes the first ad without checking the array.**
  `adCount: 1` makes that safe in this sample, but the callback signature is
  `Array<advertising.Advertisement>`.
- **`AdView`'s `onStatusChanged` takes one parameter where the document's snippet
  takes three** (`status`, `ad`, `data`). Both compile; the extra two carry the
  ad and payload if you need them.
- **`Index.restartApp` is demo-only.** `applicationContext.restartApp` exists to
  re-run the splash in the other mode; it has nothing to do with ad integration.

## References

- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-advertising -
  `advertising.AdLoader`, `loadAd`, `AdRequestParams`, `AdOptions`,
  `AdLoadListener`, `Advertisement` and `AdComponent`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/ads-publisher-service-splash -
  the splash-ad integration guide, including the AGC prerequisites and the OAID /
  tracking-consent flow.
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-f.md` -
  `window.getLastWindow` and its error-checking example.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-f
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `setWindowLayoutFullScreen`, `setPreferredOrientation`, `window.Orientation`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-router.md` -
  `replaceUrl` and `router.RouterMode`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-router
- `documentation/harmonyos-references/02_application-framework/js-apis-data-preferences.md` -
  `getPreferences`, `getSync`, `putSync`, `flushSync`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-data-preferences
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/splash_page_ad_access-0000002322822425
