---
id: COMMON-31
title: Full-application page inside a two-column layout - switch NavigationMode per destination with a route interceptor
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/31_nav_mode_change.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/nav_mode_change-0000002298406768
sample: huawei_industry_tree/19_common_technical_solutions/downloads/NavModeChange.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: [Navigation, NavigationMode, "Navigation.mode", "Navigation.navBarWidth", NavPathStack, "NavPathStack.setInterception", "NavPathStack.pushPath", "NavPathStack.pop", "NavPathStack.clear", "NavPathStack.getAllPathName", NavDestinationContext, routerMap, "UIContext.getUIObserver", "UIObserver.on('navDestinationUpdate')", "UIObserver.off", "uiObserver.NavDestinationState", "mediaquery.matchMediaSync", "@StorageLink", "@Watch", "@Provide", "@Consume", "FocusController.requestFocus", Tabs, TabsController]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0095, HW-19-0096, HW-19-0097, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when an application uses the **two-column (Split) Navigation
layout on wide screens** but one particular destination must take over the whole
window instead of appearing in the detail column - a verification screen, a
payment confirmation, a full-screen media viewer. The document's example: a chat
list in the left column, but the passcode page that guards a conversation must
cover everything.

The mechanism is that `NavigationMode` is a bindable attribute, so it can be
changed per destination - the difficulty is changing it at the right moment, and
changing it back.

## Feature checklist

The application must:

- Bind `Navigation.mode` to state rather than fixing it, and drive `navBarWidth`
  from the current breakpoint.
- Intercept navigation before the destination appears, with
  `NavPathStack.setInterception`.
- Redirect the guarded destination to the verification page and switch the mode
  to `Stack` in the same interception, so the verification page is laid out
  full-width from its first frame.
- Restore `Split` when the verification page leaves, but only on wide
  breakpoints - observed with `navDestinationUpdate` /
  `ON_WILL_DISAPPEAR`.
- Keep a placeholder page in the detail column while in Split mode so the mode
  switch does not animate an empty column - and keep it **out** of the stack on
  narrow screens (HW-19-0096).
- Mark the current page so a breakpoint change during verification does not
  switch the mode back underneath it.

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup and insets; registers the `navDestinationUpdate` observer and releases it in `onWindowStageDestroy` |
| `constant/BreakpointConstant.ets` | breakpoint names and the `CURRENT_BREAKPOINT` AppStorage key |
| `constant/CommonConstant.ets` | layout constants |
| `util/BreakpointSystem.ets` | media-query registration that publishes the current breakpoint |
| `model/DataModel.ets` | `Conversation` and the message data |
| `views/ConversationView.ets` | the message list shown in the nav bar column |
| `pages/HomePage.ets` | the `Navigation` host: owns the `NavPathStack`, the interceptor, the breakpoint watcher and the mode state |
| `pages/VerifyPage.ets` | the six-box passcode page |
| `pages/ConversationPage.ets` | the guarded destination |
| `pages/NonePage.ets` | the placeholder that fills the detail column |
| `resources/base/profile/route_map.json` | the three destinations |

**Shared state, all in AppStorage** so the ability-level observer and the page
can both reach it:

| Key | Meaning |
| --- | --- |
| `navMode` | the current `NavigationMode`, bound to `Navigation.mode` |
| `verified` | whether the passcode has been entered |
| `currentPage` | a marker set to `'VerifyPage'` while verification is showing |
| `CURRENT_BREAKPOINT` | `sm` / `md` / `lg` from the media-query system |

**The interception is the core.** `setInterception({ willShow })` runs before the
destination is shown, so the mode change lands before layout:

```
user taps a conversation
  -> pushPath('ConversationPage')
  -> willShow sees name === 'ConversationPage' && !verified
       -> pop()                       (undo the push)
       -> pushPath('VerifyPage')      (redirect)
       -> currentPage = 'VerifyPage'  (freeze the mode)
       -> navMode = NavigationMode.Stack
  -> VerifyPage lays out full-width
```

**Restoring Split** cannot happen in the page, because the page is being
destroyed. It is done from the ability-level observer, which watches for
`VerifyPage` reaching `ON_WILL_DISAPPEAR` and only restores when the breakpoint
is not `sm`.

**Why `NonePage` exists.** In Split mode the detail column must contain
something; pushing a blank placeholder avoids the mode transition animating an
empty column, which the document describes as 避免导航栏显示模式切换时的动画效果导致
页面抖动异常 ("avoid the page jitter caused by the animation when the navigation
display mode switches"). On narrow screens the placeholder would cover the home
page, so `breakpointChange` pops it - see HW-19-0096 for the case it misses.

**Layout per breakpoint.** `navBarWidth` uses a `BreakPointType` map -
full width at `sm`, half at `md` and `lg` - so the same `Navigation` is a single
column on a phone and two columns on a tablet.

## Implementation steps

1. **Bind the mode**: `@StorageLink('navMode') navMode: NavigationMode` and
   `.mode(this.navMode)` on the `Navigation`.
2. **Bind the nav bar width to the breakpoint** with a `BreakPointType` map.
3. **Register the media-query system** in `aboutToAppear` and unregister it in
   `aboutToDisappear`.
4. **Install the interceptor** in `aboutToAppear`:
   `pathStack.setInterception({ willShow: (_from, to, _operation, _isAnimated) => ... })`.
   Return early when `to === 'navBar'`; otherwise cast to
   `NavDestinationContext` and inspect `pathInfo.name` (HW-19-0095).
5. **Redirect and switch** inside the interception: `pop()`, `pushPath` the
   verification page with the original `param`, set the `currentPage` marker, set
   `navMode = NavigationMode.Stack`.
6. **Observe the exit** from the ability:
   `uiContext.getUIObserver().on('navDestinationUpdate', (info) => { ... })`,
   matching on the page name plus `ON_WILL_DISAPPEAR` plus a non-`sm`
   breakpoint, and release it in `onWindowStageDestroy`.
7. **Clear the marker** in the verification page's `aboutToDisappear`.
8. **Manage the placeholder** on breakpoint change: pop it at `sm`, push it at
   wider breakpoints when the stack is empty - guarding both branches by
   breakpoint (HW-19-0096).
9. **On success**, `pathStack.clear()`, push the real destination and set
   `verified` so the interceptor stops redirecting.

## Verified snippets

All snippets below come from the sample project, not from the document.

### The interceptor

`NavModeChange.zip#NavModeChange/entry/src/main/ets/pages/HomePage.ets`

```ts
registerInterception() {
  this.pathStack.setInterception({
    willShow: (_from, to, _operation, _isAnimated) => {
      hilog.info(0x0000, 'testTag', `interception isAnimated: ${_isAnimated}`);
      if (to === 'navBar') {
        return;
      }
      let toInfo = to as NavDestinationContext;
      let pathInfo = toInfo.pathInfo;
      // 进入聊天页前需要经过认证，这里将其转至认证页，同时调整Navigation-mode为NavigationMode.Stack，确保认证页能够占满全屏
      if (pathInfo && pathInfo.name === 'ConversationPage' && !this.verified) {
        this.pathStack.pop();
        this.pathStack.pushPath({ name: 'VerifyPage', param: pathInfo.param });
        // 这里设置当前页为VerifyPage的标记，用于避免屏幕宽度发生变化时引起NavigationMode切换从而导致认证页无法占满整个屏幕
        AppStorage.setOrCreate('currentPage', 'VerifyPage');
        this.navMode = NavigationMode.Stack;
      }
    }
  });
}
```

Note the parameter list: only the genuinely unused parameters carry the
underscore prefix. The document's copy underscores `to` as well and then uses it
(HW-19-0095).

### The Navigation host

`NavModeChange.zip#NavModeChange/entry/src/main/ets/pages/HomePage.ets`

```ts
@Entry
@Component
struct HomePage {
  @Provide('pathStack') pathStack: NavPathStack = new NavPathStack();
  @StorageLink(BreakpointConstant.CURRENT_BREAKPOINT) @Watch('breakpointChange') currentBreakpoint: string =
    BreakpointConstant.BREAKPOINT_SM;
  @StorageLink('navMode') navMode: NavigationMode = NavigationMode.Stack;
  @StorageLink('verified') verified: boolean = false;
  private breakpointSystem: BreakpointSystem = new BreakpointSystem();

  aboutToAppear(): void {
    // 初始化媒体监听设置
    this.breakpointSystem.register(this.getUIContext());
    // 注册拦截器
    this.registerInterception();
  }

  aboutToDisappear(): void {
    this.breakpointSystem.unregister();
  }

  build() {
    Navigation(this.pathStack) {
      Column() {
        this.tabsBuilder();
        this.customBarBuilder();
      }
      .height(CommonConstant.FULL_LEN)
      .width(CommonConstant.FULL_LEN);
    }
    .navBarWidth(new BreakPointType({
      sm: CommonConstant.FULL_LEN,
      md: CommonConstant.HALF_LEN,
      lg: CommonConstant.HALF_LEN
    }).getValue(this.currentBreakpoint))
    .hideTitleBar(true)
    .hideToolBar(true)
    .mode(this.navMode);
  }
}
```

### Placeholder management on breakpoint change (as shipped - see HW-19-0096)

`NavModeChange.zip#NavModeChange/entry/src/main/ets/pages/HomePage.ets`

```ts
breakpointChange() {
  // 在首页监听到断点变化，需要判断当前路由栈最顶层页面是否为NonePage，如果是NonePage，需要清除，避免SM断点下NonePage覆盖在首页上
  let pathNames = this.pathStack.getAllPathName();
  if (this.currentBreakpoint === BreakpointConstant.BREAKPOINT_SM && pathNames[pathNames.length-1] === 'NonePage') {
    this.pathStack.pop(false);
  } else if (pathNames.length === 0) {          // FIX: guard by breakpoint too
    this.pathStack.pushPath({ name: 'NonePage' });
  }
}
```

### Restoring Split from the ability-level observer

`NavModeChange.zip#NavModeChange/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
this.uiObserver = uiContext.getUIObserver();
uiContext.getUIObserver().on('navDestinationUpdate', (info) => {
  let currentBreakpoint = AppStorage.get<string>(BreakpointConstant.CURRENT_BREAKPOINT);
  if (info.name.toString() === 'VerifyPage' && info.state === uiObserver.NavDestinationState.ON_WILL_DISAPPEAR &&
    currentBreakpoint !== BreakpointConstant.BREAKPOINT_SM) {
    AppStorage.setOrCreate('navMode', NavigationMode.Split);
  }
});

onWindowStageDestroy(): void {
  // Main window is destroyed, release UI related resources
  if (this.uiObserver) {
    this.uiObserver.off('navDestinationUpdate');
  }
}
```

### The verification page

`NavModeChange.zip#NavModeChange/entry/src/main/ets/pages/VerifyPage.ets`

```ts
@Component
export struct VerifyPage {
  @Consume('pathStack') pathStack: NavPathStack;
  @State input: string[] = ['', '', '', '', '', ''];
  @StorageLink('verified') verified: boolean = false;
  param: Conversation | undefined = undefined;

  aboutToDisappear(): void {
    // 退出认证页时清空标记，避免导航栏显示状态错误切换
    AppStorage.setOrCreate('currentPage', 'OtherPage');
  }

  // ... inside each TextInput's onChange:
  if (value.length === 1) {
    if (index === MAX_INDEX) {
      // 输入到最后一位密码，清空路由栈并路由至聊天页
      this.input[index] = value;
      this.pathStack.clear();
      this.pathStack.pushPath({ name: 'ConversationPage', param: this.param });
      // 设置verified=true，避免下次路由至聊天页时被二次拦截
      AppStorage.setOrCreate('verified', true);
    } else {
      // 每输入一位密码，自动聚焦到下一个密码输入框
      this.input[index] = value;
      this.getUIContext().getFocusController().requestFocus(`PassInput-${index + 1}`);
      this.inputIndex = index + 1;
    }
  }
}
```

Note the `clear()` before `pushPath`: it removes the `VerifyPage` entry so the
back gesture from the conversation does not return to the passcode screen.

### The route table

`NavModeChange.zip#NavModeChange/entry/src/main/resources/base/profile/route_map.json`

```json
{
  "routerMap": [
    { "name": "ConversationPage", "buildFunction": "conversationPageBuilder", "pageSourceFile": "src/main/ets/pages/ConversationPage.ets" },
    { "name": "VerifyPage",       "buildFunction": "verifyPageBuilder",       "pageSourceFile": "src/main/ets/pages/VerifyPage.ets" },
    { "name": "NonePage",         "buildFunction": "nonePageBuilder",         "pageSourceFile": "src/main/ets/pages/NonePage.ets" }
  ]
}
```

## Permissions & config

**No permissions are required** and none are declared - the feature is layout and
navigation only.

`NavModeChange.zip#NavModeChange/entry/src/main/module.json5` declares
`"deviceTypes": ["phone", "tablet", "2in1"]` - the wide-screen device types are
what make the Split mode reachable - the usual `EntryAbility`, and the
`routerMap` profile above. There is no `requestPermissions` block.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`willShow` runs before the destination is shown**, which is what makes it the
  right place to change the mode - doing it in the destination's
  `aboutToAppear` would show one frame in the wrong layout.
- **`to` can be the string `'navBar'`.** Every interception must handle that case
  before casting to `NavDestinationContext`.
- **Redirecting from inside `willShow` means `pop()` then `pushPath()`** - the
  interception cannot rewrite the pending navigation in place.
- **The mode must be restored from outside the page.** The page that triggered
  `Stack` is gone by the time `Split` should return, so the observer lives at
  ability level - and must be released in `onWindowStageDestroy`.
- **`ON_WILL_DISAPPEAR` rather than `ON_DISAPPEAR`**: the mode has to change while
  the exit transition is still running, otherwise the column layout snaps after
  the animation.
- **`NavigationMode.Auto` is not used here.** The whole point is that the mode is
  a decision the application makes per destination, so it is pinned to `Stack` or
  `Split` explicitly.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **The document's interception snippet does not compile.** It declares `_to` and
  then uses `to`; the sample names the parameter `to`. (HW-19-0095)
- **`breakpointChange` pushes `NonePage` whenever the stack is empty, including at
  the `sm` breakpoint, which is incorrect** - the placeholder then covers the home
  page, which is exactly what the method's own comment says to avoid. Guard the
  second branch by breakpoint. (HW-19-0096)
- **The top inset comes from `TYPE_CUTOUT` plus a hardcoded 20, which is
  incorrect**, and the `avoidAreaChange` subscription is never released. On a
  tablet or 2in1 without a cutout the inset degenerates to a flat 20 vp.
  (HW-19-0097)
- **`currentPage` is written but never read in the shipped code.** The comment
  explains its purpose - stopping a breakpoint change from switching the mode
  while verification is showing - but no code consumes it; the observer guards on
  the page name instead. Either wire it up or remove it.
- **`verified` is a single global flag in AppStorage.** Once true, every
  conversation opens without verification for the rest of the session, and it is
  not persisted, so it resets on restart.
- **`pathStack.pop()` inside `willShow` fires the interception again.** It works
  here because the popped destination is not `ConversationPage`, but an
  interceptor that redirects to a name it also intercepts would recurse.

## References

- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-basic-components-navigation -
  `Navigation`, `NavigationMode` (Stack / Split / Auto), `mode`, `navBarWidth`,
  `NavPathStack.setInterception`, `pushPath`, `pop`, `clear`,
  `getAllPathName`, and `NavDestinationContext`.
- `documentation/harmonyos-references/02_application-framework/js-apis-arkui-observer.md` -
  `UIObserver.on('navDestinationUpdate')` / `off`, `NavDestinationState`
  including `ON_WILL_DISAPPEAR`.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-arkui-observer
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-e.md` -
  `AvoidAreaType.TYPE_SYSTEM` vs `TYPE_CUTOUT`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-e
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-split-mode.md` -
  the Split-mode layout this sample deviates from for one destination.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-split-mode
- `documentation/harmonyos-guides/03_application-framework/arkts-appstorage.md` -
  `@StorageLink` and `@Watch`, which connect the breakpoint system to the page.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-appstorage
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/nav_mode_change-0000002298406768
