---
id: COMMON-12
title: Custom theme colours - swap component colour tokens at runtime with CustomTheme and WithTheme
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/12_custom_theme_demo.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_theme_demo-0000002290124205
sample: huawei_industry_tree/19_common_technical_solutions/downloads/CustomThemeDemo.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: [CustomTheme, CustomColors, Colors, WithTheme, ThemeControl, "display.getDefaultDisplaySync", "WindowStage.createSubWindow", "Window.setWindowLayoutFullScreen", "Window.setWindowTouchable", "Window.setUIContent", "Window.setWindowBackgroundColor", "Window.showWindow", "Window.destroyWindow", "Window.moveWindowTo", "Window.setWindowBrightness", "ApplicationContext.setColorMode", "ConfigurationConstant.ColorMode", "@StorageProp", "PromptAction.showToast", "UIContext.px2vp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0021, HW-19-0022, HW-19-0023, HW-19-0024]
status: verified-with-fixes
---

## When to use

Load this card when the product lets the user **pick an accent colour / theme for
the application itself**, and the change must apply to system components
(buttons, sliders, radio indicators, emphasised backgrounds) without every
component having to be styled by hand.

The mechanism is the ArkUI theme system: define a `CustomTheme` whose `colors`
override the standard colour tokens, then wrap the subtree in `WithTheme({ theme
})`. Switching the theme is a single state assignment.

The sample dresses this up as a system "Display" settings page, so it also
demonstrates screen brightness control, an eye-protection overlay implemented as
a non-touchable full-screen child window, and blocking the system dark mode.

## Feature checklist

The application must:

- Define a `CustomColors` implementation listing the colour tokens to override.
- Wrap that in a `CustomTheme` implementation and expose a shared instance.
- Also expose an "empty" `CustomTheme` to mean "use the system theme".
- Hold the current theme in a state variable and wrap the page content in
  `WithTheme({ theme })`.
- Switch themes by assigning the state variable - no reload, no restart.
- Keep the non-theme-aware parts of the UI (custom images, explicit
  `fontColor(...)` values) in sync with the same selection flag.
- Optionally: drive window brightness, toggle an eye-protection overlay, and
  force/release light colour mode.

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | publishes display width/height and the `WindowStage` into `AppStorage`, sets full-screen layout, publishes and maintains the two insets, loads `pages/DisplayPage` |
| `common/AppColors.ets` | `AppColors implements CustomColors`, `AppTheme implements CustomTheme` (with those colours), and `CustomTheme1 implements CustomTheme` (empty = system); exports the two singletons `gAppTheme` and `gCustomTheme1` |
| `constants/StyleConstants.ets` | layout constants, the setting-row indices, the eye-protection colour `#4d255f1e`, and `TOGGLE_LIST` |
| `components/ThemeItem.ets` | one theme card: preview image, title, radio dot |
| `components/SubWindow.ets` | the (empty) `@Entry` page loaded into the eye-protection child window |
| `util/EyeProtectionMode.ets` | singleton that creates and destroys the eye-protection child window |
| `pages/DisplayPage.ets` | the settings page: `WithTheme` wrapper, theme cards, brightness slider, four toggles |

**How the theme swap works.** `AppColors` sets five tokens - `fontPrimary`,
`fontOnPrimary`, `iconOnPrimary`, `compBackgroundEmphasize`,
`backgroundEmphasize` - to a single custom colour resource. `AppTheme` exposes
them as its `colors`. `CustomTheme1` overrides nothing, so it is the system
default. `DisplayPage` holds `@State myTheme: CustomTheme = gCustomTheme1` and
passes it to `WithTheme`; tapping a theme card assigns `gAppTheme` or
`gCustomTheme1` and the whole subtree recolours.

A second flag, `@State isCustomTheme: boolean`, drives everything the theme
system does **not** reach: the card titles, the radio dot images and the slider
block border colour, which are set with explicit resources rather than tokens.
Both flags are written together in each `onClick`.

**The eye-protection overlay** is a separate mechanism: a child window named
`eyeWindow`, moved to (0,0), laid out full-screen, made **non-touchable** so it
does not swallow gestures on the main window, loaded with an empty page and given
a translucent warm background colour. Disabling destroys the window.

## Implementation steps

1. **Declare the colour overrides.**
   `export class AppColors implements CustomColors { fontPrimary: ResourceColor =
   $r('app.color.custom_color'); ... }`. Import `CustomColors` and `CustomTheme`
   from `@kit.ArkUI` (HW-19-0021).
2. **Wrap them in themes and export singletons.** One theme with `colors`, one
   empty theme meaning "system". Export module-level instances so the identity is
   stable across renders.
3. **Hold the active theme in `@State`** and wrap the page body in
   `WithTheme({ theme: this.myTheme })`.
4. **Add a parallel boolean** for the non-tokenised visuals, and set both in the
   same click handler.
5. **Publish the window stage and display metrics** in `onWindowStageCreate`
   before `loadContent`, so the singletons and the child-window page can read
   them from `AppStorage`.
6. **Consume the insets without mutating them.** Keep `@StorageProp` values in
   their original px unit and convert at the point of use (HW-19-0024).
7. **Brightness:** `windowStage.getMainWindowSync().setWindowBrightness(value /
   100)` from the slider's `onChange`.
8. **Eye protection:** create the child window, check the callback error before
   using the result (HW-19-0022), move it to the origin, set full-screen and
   `setWindowTouchable(false)`, load a blank page, set a translucent background
   colour in the `setUIContent` callback, then `showWindow`. On disable,
   `destroyWindow`, check the error, and null the handle (HW-19-0023).
9. **Dark-mode override:**
   `context.getApplicationContext().setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT)`
   to block it and `COLOR_MODE_NOT_SET` to release it.

## Verified snippets

All snippets below come from the sample project, not from the document.

### Theme definitions

`CustomThemeDemo.zip#CustomThemeDemo/entry/src/main/ets/common/AppColors.ets`

```ts
import { CustomColors, CustomTheme } from '@ohos.arkui.theme'; // FIX (HW-19-0021): use '@kit.ArkUI'

export class AppColors implements CustomColors {
  fontPrimary: ResourceColor = $r('app.color.custom_color');
  fontOnPrimary: ResourceColor = $r('app.color.custom_color');
  iconOnPrimary: ResourceColor = $r('app.color.custom_color');
  compBackgroundEmphasize: ResourceColor = $r('app.color.custom_color');
  backgroundEmphasize: ResourceColor = $r('app.color.custom_color');
}

export class CustomTheme1 implements CustomTheme {
}

export const gCustomTheme1: CustomTheme = new CustomTheme1();

export class AppTheme implements CustomTheme {
  public colors: AppColors = new AppColors();
}

export const gAppTheme: CustomTheme = new AppTheme();
```

### Applying and switching the theme

`CustomThemeDemo.zip#CustomThemeDemo/entry/src/main/ets/pages/DisplayPage.ets`

```ts
import { CustomTheme, PromptAction, window } from '@kit.ArkUI';
import { gAppTheme, gCustomTheme1 } from '../common/AppColors';

@Entry
@Component
export struct DisplayPage {
  @StorageProp('topRectHeight') topRectHeight: number = StyleConstants.ZERO;
  @State myTheme: CustomTheme = gCustomTheme1;
  @State isCustomTheme: boolean = false;
  windowStage: window.WindowStage = AppStorage.get('windowStage') as window.WindowStage;
  mainWin: window.Window = this.windowStage.getMainWindowSync();

  build() {
    WithTheme({ theme: this.myTheme }) {
      Column({ space: StyleConstants.INDEX_SPACE }) {
        Row() {
          ThemeItem({
            img: $r('app.media.img_system'),
            title: StyleConstants.SYSTEM_THEME,
            color: this.isCustomTheme ? $r('app.color.unselect_color') : $r('app.color.system_color'),
            radio: this.isCustomTheme ? $r('app.media.dot') : $r('app.media.system_dot')
          })
            .onClick(() => {
              this.isCustomTheme = false;
              this.myTheme = gCustomTheme1;
            });
          ThemeItem({
            img: $r('app.media.img_custom'),
            title: StyleConstants.CUSTOM_THEME,
            color: this.isCustomTheme ? $r('app.color.custom_color') : $r('app.color.unselect_color'),
            radio: this.isCustomTheme ? $r('app.media.custom_dot') : $r('app.media.dot')
          })
            .onClick(() => {
              this.isCustomTheme = true;
              this.myTheme = gAppTheme;
            });
        };
        // ... brightness slider and setting rows ...
      }
      .padding({ top: this.topRectHeight }); // FIX (HW-19-0024): px2vp here, not in aboutToAppear
    };
  }
}
```

### Brightness

`CustomThemeDemo.zip#CustomThemeDemo/entry/src/main/ets/pages/DisplayPage.ets`

```ts
aboutToAppear(): void {
  // 修改brightness即可改变屏幕亮度
  let brightness = this.brightnessValue / StyleConstants.BRIGHTNESS_MAX;
  this.mainWin.setWindowBrightness(brightness);
}

Slider({ value: this.brightnessValue, max: this.brightnessMax })
  .blockBorderColor(this.isCustomTheme ? $r('app.color.custom_color') : $r('app.color.system_color'))
  .onChange((value) => {
    this.brightnessValue = value;
    this.mainWin.setWindowBrightness(value / StyleConstants.BRIGHTNESS_MAX);
  });
```

### Eye-protection overlay as a non-touchable child window

`CustomThemeDemo.zip#CustomThemeDemo/entry/src/main/ets/util/EyeProtectionMode.ets`

```ts
export class EyeProtectionMode {
  windowStage: window.WindowStage = AppStorage.get('windowStage') as window.WindowStage;
  eyeWindowClass: window.Window | null = null;
  private static sInstance: EyeProtectionMode;

  public static getInstance(): EyeProtectionMode {
    if (!EyeProtectionMode.sInstance) {
      EyeProtectionMode.sInstance = new EyeProtectionMode();
    }
    return EyeProtectionMode.sInstance;
  }

  createSubWithEyeWindow(backgroundColor: string): void {
    // 创建护眼模式子窗口
    this.windowStage.createSubWindow('eyeWindow', (error: BusinessError, data) => {
      // FIX (HW-19-0022): if (error.code) { LOGGER.error(...); return; }
      this.eyeWindowClass = data;
      this.eyeWindowClass.moveWindowTo(0, 0);
      // 设置全屏显示
      this.eyeWindowClass?.setWindowLayoutFullScreen(true);
      // 设置不可触摸，防止子窗口阻挡主窗口的手势
      this.eyeWindowClass.setWindowTouchable(false);
      this.eyeWindowClass.setUIContent('components/SubWindow', (err: BusinessError) => {
        // 设置背景颜色
        if (!err.code) {
          this.eyeWindowClass?.setWindowBackgroundColor(backgroundColor);
        }
      });
      // 显示窗口
      this.eyeWindowClass?.showWindow(() => {
      });
    });
  }

  // 移除护眼模式子窗口
  removeSubWithEyeWindow(): void {
    if (this.eyeWindowClass != null) {
      this.eyeWindowClass.destroyWindow(() => {
        // FIX (HW-19-0023): check err.code and set this.eyeWindowClass = null here
      });
    }
  }
}
```

The overlay colour is a constant:
`CustomThemeDemo.zip#CustomThemeDemo/entry/src/main/ets/constants/StyleConstants.ets`

```ts
static readonly EYE_PROTECTION_COLOR: string = '#4d255f1e';
```

### Toggle dispatch

`CustomThemeDemo.zip#CustomThemeDemo/entry/src/main/ets/pages/DisplayPage.ets`

```ts
handleOnclick(id: number) {
  switch (id) {
    case StyleConstants.EYE_PROTECTION:
      if (this.toggleList[StyleConstants.EYE_PROTECTION]) {
        EyeProtectionMode.getInstance().createSubWithEyeWindow(StyleConstants.EYE_PROTECTION_COLOR);
      } else {
        EyeProtectionMode.getInstance().removeSubWithEyeWindow();
      }
      break;
    case StyleConstants.DISABLE_COLOR_MODE:
      if (this.toggleList[StyleConstants.DISABLE_COLOR_MODE]) {
        this.context.getApplicationContext().setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT);
      } else {
        this.context.getApplicationContext().setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_NOT_SET);
        this.promptAction.showToast({ message: $r('app.string.close_color_mode') });
      }
      break;
    // ...
  }
}
```

### Publishing the stage and display metrics

`CustomThemeDemo.zip#CustomThemeDemo/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
let displayClass: display.Display = display.getDefaultDisplaySync();
AppStorage.setOrCreate('windowWidth', displayClass.width);
AppStorage.setOrCreate('windowHeight', displayClass.height);
AppStorage.setOrCreate('windowStage', windowStage);
let windowClass: window.Window = windowStage.getMainWindowSync();
windowClass.setWindowLayoutFullScreen(true).then(() => { /* ... */ });
// avoid areas published and kept current via windowClass.on('avoidAreaChange', ...)
windowStage.loadContent('pages/DisplayPage', (err) => { /* ... */ });
```

## Permissions & config

**No permissions are required.** The shipped `module.json5` declares no
`requestPermissions` block. `setWindowBrightness` affects only this
application's own window, and the child window is created through the
application's own `WindowStage`.

`CustomThemeDemo.zip#CustomThemeDemo/entry/src/main/module.json5`:

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
    ]
  }
}
```

Note that `components/SubWindow` is loaded by `setUIContent` as a page path, so it
must be reachable as a page even though it lives under `ets/components/`.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. The theme module itself dates from
  API 12: "The initial APIs of this module are supported since API version 12."
- **`CustomTheme` only covers the documented colour tokens.** Anything the app
  colours explicitly - the images, the card titles, the slider block border in
  this sample - is untouched by `WithTheme` and needs its own state, which is why
  the sample carries `isCustomTheme` alongside `myTheme`.
- **A child window name is unique per stage.** `createSubWindow` fails with
  1300002 "This window state is abnormal. Possible cause: The subWindow has been
  created and can not be created again."
- **Child windows are immersive by default**: "The child window created uses an
  immersive layout by default."
- **`setWindowTouchable(false)` is required for an overlay**, otherwise the
  overlay swallows every gesture aimed at the main window.
- **`@StorageProp` is one-way and overwritable**: "any local changes will be
  overwritten" when the AppStorage key changes.
- **Devices.** `phone`, `tablet`, `2in1`; the theme module is available on Phone,
  PC/2in1, Tablet, TV and Wearable.

## Pitfalls

- **`import { CustomColors, CustomTheme } from '@ohos.arkui.theme'` is not the
  documented form.** The reference prescribes `from '@kit.ArkUI'`, and the sibling
  file `DisplayPage.ets` in the same project already does it that way.
  (HW-19-0021)
- **The `createSubWindow` callback ignores its error and dereferences `data`,
  which is incorrect.** The official example reads `err.code` and returns before
  using the window; without that, the documented 1300002 failure becomes a crash.
  (HW-19-0022)
- **`removeSubWithEyeWindow` leaves a stale handle, which is incorrect.**
  `destroyWindow` is called with an empty callback and `eyeWindowClass` is never
  nulled, so the null guard still passes on the next disable and the previous
  window object leaks if enable is called twice. Clear the handle in the callback
  after checking the error. (HW-19-0023)
- **Converting a `@StorageProp` in place is incorrect.**
  `this.topRectHeight = px2vp(AppStorage.get('topRectHeight'))` is discarded the
  next time `avoidAreaChange` writes the key, leaving a px value where a vp value
  is expected. Convert at the point of use. (HW-19-0024)
- **`Toggle` is wired with `onClick`, not `onChange`,** and `toggleList` is a
  plain member aliasing the exported `TOGGLE_LIST` constant rather than a `@State`
  array. The switch works because `Toggle` keeps its own visual state, but the
  data model and the UI are not actually bound - do not copy this shape into code
  that needs to read the toggle state back.
- **The empty `CustomTheme1` is the "system theme" sentinel.** It looks like dead
  code; it is not.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-arkui-theme.md` -
  `Theme`, `Colors`, `CustomColors`, `CustomTheme`, `ThemeControl`, the
  documented import from `@kit.ArkUI`, and the API 12 baseline.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-arkui-theme
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-container-with-theme -
  the `WithTheme` container component.
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-windowstage.md` -
  `createSubWindow` (callback and promise forms), the error-checking example, the
  immersive-by-default note and error 1300002.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-windowstage
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `moveWindowTo`, `setWindowTouchable`, `setUIContent`,
  `setWindowBackgroundColor`, `showWindow`, `destroyWindow`,
  `setWindowBrightness`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-guides/03_application-framework/arkts-appstorage.md` -
  `@StorageProp` one-way synchronisation and the overwrite rule.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-appstorage
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_theme_demo-0000002290124205
