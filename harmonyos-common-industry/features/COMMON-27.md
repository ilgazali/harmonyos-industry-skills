---
id: COMMON-27
title: Blur the home page until login - foregroundBlurStyle gated on the login state, with a login prompt on top
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/27_display_after_login.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/display_after_login-0000002293948070
sample: huawei_industry_tree/19_common_technical_solutions/downloads/DisplayAfterLogin.zip
kits: ["@kit.ArkUI"]
apis: ["component .foregroundBlurStyle", BlurStyle, ThemeColorMode, AdaptiveColor, "component .enabled", "component .hitTestBehavior", HitTestMode, Navigation, NavPathStack, "NavPathStack.pushDestinationByName", routerMap, Stack, Swiper, "@Provide", "@Consume", "@StorageLink", ListItem]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0084, HW-19-0085, HW-19-0086, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when the home screen should be **visible but unreadable until the
user signs in** - the content is blurred behind a login prompt, and becomes crisp
and usable the moment the login state flips. It is a softer alternative to
routing an unauthenticated user to a login page: the user sees that there is
content worth signing in for.

The whole technique is one attribute, `foregroundBlurStyle`, bound to a boolean.
The part the sample gets wrong is that blurring is not gating - see HW-19-0084.

## Feature checklist

The application must:

- Hold the login state where both the home page and the login destination can
  reach it.
- Apply `foregroundBlurStyle` to the content subtree, choosing `BlurStyle.NONE`
  when logged in and a real blur style otherwise.
- **Also block interaction** with that subtree while logged out (HW-19-0084).
- Overlay a login prompt on the blurred content, with a button that navigates to
  the login destination.
- Clear both the blur and the prompt when the login destination sets the state.

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup; publishes `topRectHeight` / `bottomRectHeight` |
| `common/DataModel.ets` | `SwiperData` and `TabData` |
| `component/SwiperComponent.ets` | `SwiperStackComponent`, the carousel on the home feed |
| `pages/MainPage.ets` | the start page: the news feed, the blur, the login prompt, and the `NavPathStack` |
| `pages/LoginPage.ets` | the login destination, exported through `loginPageBuilder` |
| `resources/base/profile/route_map.json` | route `LoginPage` -> `loginPageBuilder` |

**Shared state.** The start page declares three `@Provide`s:

```ts
@Provide isLogin: boolean = false;      // the gate
@Provide pageInfos: NavPathStack = new NavPathStack();
@Provide isShow: boolean = true;        // whether the login prompt is visible
```

`LoginPage` takes `@Consume isLogin` and sets it to `true` on a successful sign
in, which propagates back and both removes the blur and hides the prompt.

**Layering.** The page is a `Stack` inside a `Navigation`:

1. The content `Column` - header, search row, carousel, feed, bottom tab bar -
   carries the `foregroundBlurStyle`.
2. `if (!this.isLogin && this.isShow) { this.loginTips(); }` puts the prompt card
   on top.

The prompt has a close button that sets `isShow = false`. That is the hole: it
removes the overlay while leaving the blur - and the interactivity - in place
(HW-19-0084).

**Navigation.** The prompt's login button calls
`this.pageInfos.pushDestinationByName('LoginPage', '', true)`. The third argument
enables the animation; `pushDestinationByName` (rather than `pushPathByName`)
returns a promise that rejects if the route name is not registered, which is the
safer choice when the destination lives in a `routerMap` profile.

## Implementation steps

1. **Provide the login state** from the start page with `@Provide`, and consume
   it in the login destination with `@Consume`.
2. **Wrap the content and the prompt in a `Stack`** so the prompt can sit above
   the blurred subtree.
3. **Apply the blur conditionally**:
   ```ts
   .foregroundBlurStyle(this.isLogin ? BlurStyle.NONE : BlurStyle.BACKGROUND_THICK, {
     colorMode: ThemeColorMode.LIGHT,
     adaptiveColor: AdaptiveColor.DEFAULT,
     scale: 0.15
   })
   ```
4. **Block interaction on the same subtree** - `.enabled(this.isLogin)` or
   `.hitTestBehavior(this.isLogin ? HitTestMode.Default : HitTestMode.Block)` -
   because the blur alone does not (HW-19-0084).
5. **Render the prompt** only while logged out, and think hard before letting the
   user dismiss it: if it can be closed, the gate must come from step 4, not
   from the overlay.
6. **Register the login destination** in `route_map.json` and push it by name.
7. **Set `isLogin = true` from the login page**; everything else follows from the
   state binding.

## Verified snippets

All snippets below come from the sample project, not from the document.

### The blur, and what it does not do

`DisplayAfterLogin.zip#DisplayAfterLogin/entry/src/main/ets/pages/MainPage.ets`

```ts
        }
        .height('100%')
        .width('100%')
        .padding({
          top: this.getUIContext().px2vp(this.topRectHeight),
          bottom: this.getUIContext().px2vp(this.bottomRectHeight),
        })
        // 根据登录态，设置组件模糊
        .foregroundBlurStyle(this.isLogin ? BlurStyle.NONE : BlurStyle.BACKGROUND_THICK, {
          colorMode: ThemeColorMode.LIGHT,
          adaptiveColor: AdaptiveColor.DEFAULT,
          scale: 0.15
        });
        // FIX (HW-19-0084): add .enabled(this.isLogin) or .hitTestBehavior(...)

        if (!this.isLogin && this.isShow) {
          this.loginTips();
        }
      }
      .height('100%')
      .width('100%');
    }
    .hideTitleBar(true)
    .hideToolBar(true);
```

### Shared state and the entry component

`DisplayAfterLogin.zip#DisplayAfterLogin/entry/src/main/ets/pages/MainPage.ets`

```ts
@Entry
@Component
struct LoginPage {            // FIX (HW-19-0086): this is the main page, not the login page
  @Provide isLogin: boolean = false;
  @Provide pageInfos: NavPathStack = new NavPathStack();
  @Provide isShow: boolean = true;
  @State currentIndex: number = 0;
  @StorageLink('topRectHeight') topRectHeight: number = 0;
  @StorageLink('bottomRectHeight') bottomRectHeight: number = 0;
  @State swiperData: SwiperData[] = [ /* five carousel entries */ ];
  @State data: TabData[] = [ /* five bottom-bar entries */ ];
}
```

### The login prompt

`DisplayAfterLogin.zip#DisplayAfterLogin/entry/src/main/ets/pages/MainPage.ets`

```ts
// 登录拦截提示
@Builder
loginTips() {
  Column({ space: 15 }) {
    Row() {
      Text($r('app.string.Login'))
        .fontSize(18)
        .lineHeight(24)
        .fontWeight(700)
        .opacity(0.9);
      Image($r('app.media.close_button'))
        .size({ width: 40, height: 40 })
        .onClick(() => {
          this.isShow = false;        // removes the overlay, leaves the blur and the taps
        });
    }
    .width(328)
    .justifyContent(FlexAlign.SpaceBetween)
    .alignSelf(ItemAlign.Start)
    .padding({ left: 16, right: 16, top: 24 });

    Text($r('app.string.To_view_more'))
      .fontSize(16)
      .lineHeight(24)
      .fontWeight(400)
      .margin({ top: 42 });

    Button($r('app.string.Login'), { type: ButtonType.Normal })
      .borderRadius(20)
      .width(296)
      .height(40)
      .margin({ top: 35, bottom: 24 })
      .backgroundColor('#0A59F7')
      .onClick(() => {
        this.pageInfos.pushDestinationByName('LoginPage', '', true);
      });
  }
  .alignItems(HorizontalAlign.Center)
  .width(328)
  .borderRadius(16)
  .backgroundColor($r('sys.color.comp_background_list_card'));
}
```

### The one control that is disabled

`DisplayAfterLogin.zip#DisplayAfterLogin/entry/src/main/ets/pages/MainPage.ets`

```ts
Search({ placeholder: $r('app.string.Search') }).layoutWeight(1).enabled(false);
```

The search box is the only component in the whole page with `.enabled(false)` -
everything else under the blur stays live.

### The bottom bar (as shipped - see HW-19-0085)

`DisplayAfterLogin.zip#DisplayAfterLogin/entry/src/main/ets/pages/MainPage.ets`

```ts
Row() {
  ForEach(this.data, (item: TabData) => {
    ListItem() {          // FIX: ListItem's parent may only be List or ListItemGroup
      this.tabItem(item);
    };
  });
}
.padding({ left: 16, right: 16 })
.height(49)
.width('100%')
.alignItems(VerticalAlign.Bottom)
.justifyContent(FlexAlign.SpaceBetween);
```

### The login destination

`DisplayAfterLogin.zip#DisplayAfterLogin/entry/src/main/ets/pages/LoginPage.ets`

```ts
export function loginPageBuilder() { /* ... */ }

@Component
export struct LoginPage {
  @Consume isLogin: boolean;
  // ...
  .onClick(() => {
    // ...
    this.isLogin = true;
  })
}
```

`DisplayAfterLogin.zip#DisplayAfterLogin/entry/src/main/resources/base/profile/route_map.json`

```json
{
  "routerMap": [
    {
      "name": "LoginPage",
      "pageSourceFile": "src/main/ets/pages/LoginPage.ets",
      "buildFunction": "loginPageBuilder",
      "data": { "description": "this is LoginPagePage" }
    }
  ]
}
```

## Permissions & config

**No permissions are required** and none are declared - the feature is a
rendering attribute plus a state variable. There is no real authentication in
this sample: the login page simply sets `isLogin = true`.

`main_pages.json` lists only `pages/MainPage`; the login screen is a Navigation
destination registered in `route_map.json`, referenced from `module.json5` as
`"routerMap": "$profile:route_map"`.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`foregroundBlurStyle` is a rendering attribute only.** It does not affect hit
  testing, focus or accessibility - a blurred subtree is still tappable and still
  readable by a screen reader.
- **`BlurStyle.NONE` is how you turn it off**; there is no "unset" at the
  attribute level, so both branches of the conditional must supply a style.
- **`scale` shrinks the blur sampling** (0.15 here), trading fidelity for cost;
  the blur is a GPU effect over the whole subtree, so keep the blurred subtree
  as small as the design allows.
- **`adaptiveColor` and `colorMode`** decide whether the blur tints from the
  content beneath and whether it uses the light or dark palette - pin them if the
  app also supports dark mode, otherwise the blur inverts with the system theme.
- **`ListItem`'s parent may only be `List` or `ListItemGroup`.**
- **The login state is in-memory only.** Nothing persists it, so every launch
  starts blurred; a real app would combine this with the silent-login pattern of
  COMMON-17.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **Treating the blur as an access gate is incorrect.** `foregroundBlurStyle`
  changes drawing, not input; the pre-login page keeps every click handler, and
  the prompt's close button removes the only overlay covering it. Pair the blur
  with `.enabled(false)` or `HitTestMode.Block`. (HW-19-0084)
- **Wrapping the tab items in `ListItem` inside a `Row` is incorrect** - the
  reference restricts `ListItem`'s parent to `List` or `ListItemGroup`. Drop the
  wrapper. (HW-19-0085)
- **`MainPage.ets` declares `struct LoginPage`, which is confusing and wrong.**
  Two components in the project share that name, and the misnamed one is the
  start page. (HW-19-0086)
- **The five bottom-bar items have no `onClick`.** The bar is decorative in this
  sample; do not read it as a working tab implementation.
- **`isShow` and `isLogin` are two separate gates.** Once `isShow` is false the
  prompt never returns, even though `isLogin` is still false - so the user has no
  way back to the login button without restarting the app.
- **The five carousel entries all point at the same image** (`app.media.ads_1`);
  the `name` field is what distinguishes them.

## References

- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-attributes-foreground-blur-style -
  `foregroundBlurStyle`, `BlurStyle`, `ThemeColorMode`, `AdaptiveColor` and the
  `scale` option.
- `documentation/harmonyos-references/02_application-framework/ts-container-listitem.md` -
  "It must be used together with **List**" and "The parent of this component can
  only be List or ListItemGroup".
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-listitem
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-hit-test-behavior.md` -
  `hitTestBehavior` / `HitTestMode`, the attribute that actually blocks input.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-hit-test-behavior
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` -
  `Navigation`, `NavPathStack.pushDestinationByName` and the `routerMap` profile.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-provide-and-consume.md` -
  `@Provide` / `@Consume`, which carry `isLogin` across the page boundary.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-provide-and-consume
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/display_after_login-0000002293948070
