---
id: COMMON-32
title: Foldable split-column adaptation - drive NavigationMode from the live window width
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/32_app_multiplier.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_multiplier-0000002312850858
sample: huawei_industry_tree/19_common_technical_solutions/downloads/APPMultiplier.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.ArkTS", "@kit.PerformanceAnalysisKit"]
apis: ["WindowStage.getMainWindow", "Window.getWindowProperties", "Window.on('windowSizeChange')", "Window.off('windowSizeChange')", "window.Size", "display.getDefaultDisplaySync", "Display.densityPixels", Navigation, NavigationMode, "Navigation.mode", NavPathStack, NavDestination, "NavDestination.onBackPressed", "@StorageLink", "AppStorage.setOrCreate", "resourceManager.getRawFileContent", "buffer.from", Tabs, List, ListItem, RelativeContainer]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0098, HW-19-0099, HW-19-0182, HW-19-0183]
status: verified-with-fixes
---

## When to use

Load this card when the application should show **one column folded and two
columns unfolded** on a foldable, and the switch must happen live as the device
is folded - not only at launch. It is the simplest possible version of the
adaptive-layout problem: one measured value, one derived breakpoint string, one
bound `NavigationMode`.

Compare with COMMON-31, which solves the opposite problem - keeping one specific
destination out of the two-column layout - and with COMMON-36, which adapts an
embedded web page rather than a native layout.

## Feature checklist

The application must:

- Obtain the main window and read its current width at startup.
- Subscribe to `windowSizeChange` so folding and unfolding are observed live.
- Convert the reported pixel width into vp before comparing it to a breakpoint.
- Publish the resulting breakpoint string into `AppStorage` so any page can bind
  to it.
- Bind `Navigation.mode` to that string: `Split` when wide, `Stack` when narrow.
- Release the subscription when the window stage is destroyed (HW-19-0098).
- Handle the failure to obtain the window (HW-19-0099).

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | the measurement half: obtains the window, reads the width, subscribes to `windowSizeChange`, and publishes `curScreenSizeString` |
| `constant/CommonConstant.ets` | the breakpoint value and the two breakpoint strings, plus the tab-bar geometry for each |
| `datastruct/DataStruct.ets` | `DisplayInfo` and `NavPathInfoParamObj` |
| `toolability/APPMultiplierTools.ets` | `getDataFromJSON` (rawfile loader), `DisplayDeviceItem`, and `buildNavDestination` |
| `component/DisplayItem.ets` | the list row in the nav-bar column |
| `pages/AppMultiplier.ets` | the `Navigation` host; binds `curScreenSizeString` and switches the mode |

**One state variable crosses the boundary.** The ability writes
`AppStorage.setOrCreate('curScreenSizeString', ...)` and the page reads it with
`@StorageLink('curScreenSizeString')`. Nothing else is shared, and the page never
touches the window API.

**The measurement.**

```ts
let realWindowWidth = windowWidth / display.getDefaultDisplaySync().densityPixels;
```

`windowSizeChange` reports the width in **px**; the breakpoint is expressed in
**vp**, so the raw width is divided by the display density. The sample's own
comment records the measured widths it was tuned against: a Mate 60 is 374 vp
portrait, an X6 folded is 345.6 vp portrait, and an X6 unfolded is 716.8 vp
portrait - which is why the threshold is 640 rather than one of the default
grid breakpoints (320 / 600 / 840 vp). It is a device-specific value chosen to
sit between folded and unfolded, and the comment block documents that choice.

**The binding.**

```ts
.mode(this.curScreenSizeString === CommonConstant.SCREEN_SIZE_STRING_LG
  ? NavigationMode.Split : NavigationMode.Stack)
```

Because the attribute reads a `@StorageLink` variable, the whole adaptation is
one re-render - there is no imperative mode switching anywhere.

**The tab bar adapts too.** `CommonConstant` carries two geometries -
`TAB_BAR_WIDTH_PERCENT` / `TAB_BAR_HEIGHT_PERCENT` for the narrow case and
`TAB_BAR_WIDTH_PERCENT_LG` / `TAB_BAR_HEIGHT_PERCENT_LG` for the wide one - so
the bar becomes a vertical rail (10% wide, 100% tall) when unfolded.

## Implementation steps

1. **Get the main window** in `onWindowStageCreate`, with a rejection handler
   (HW-19-0099).
2. **Read the initial width** from
   `windowObj.getWindowProperties().windowRect.width` - the subscription only
   reports *changes*, so the first value must be read explicitly.
3. **Subscribe**: `windowObj.on('windowSizeChange', (windowSize: window.Size) =>
   this.updateScreenDisplay(windowSize.width))`.
4. **Convert px to vp** with `display.getDefaultDisplaySync().densityPixels`
   before comparing against the breakpoint.
5. **Publish the breakpoint string** into `AppStorage`.
6. **Bind it in the page** with `@StorageLink` and use it in `.mode(...)` and in
   any layout that differs between the two states.
7. **Release the subscription** in `onWindowStageDestroy` (HW-19-0098).

## Verified snippets

All snippets below come from the sample project, not from the document.

### Measuring and publishing

`APPMultiplier.zip#APPMultiplier/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
onWindowStageCreate(windowStage: window.WindowStage): void {
  hilog.info(DOMAIN, 'testTag', '%{public}s', 'Ability onWindowStageCreate');

  windowStage.getMainWindow().then((data: window.Window) => {
    // 监听屏幕尺寸
    this.windowObj = data;
    this.updateScreenDisplay(this.windowObj.getWindowProperties().windowRect.width);
    this.windowObj.on('windowSizeChange', (windowSize: window.Size) => {
      this.updateScreenDisplay(windowSize.width);
    });
  });
  // FIX (HW-19-0099): .catch((err: BusinessError) => hilog.error(...))

  windowStage.loadContent('pages/AppMultiplier', (err) => {
    if (err.code) {
      hilog.error(DOMAIN, 'testTag', 'Failed to load the content. Cause: %{public}s', JSON.stringify(err));
      return;
    }
    hilog.info(DOMAIN, 'testTag', 'Succeeded in loading the content.');
  });
}

onWindowStageDestroy(): void {
  // Main window is destroyed, release UI related resources
  // FIX (HW-19-0098): this.windowObj?.off('windowSizeChange');
  hilog.info(DOMAIN, 'testTag', '%{public}s', 'Ability onWindowStageDestroy');
}

private updateScreenDisplay(windowWidth: number): void {
  // windowWidth屏幕宽度像素，realWindowWidth屏幕真实宽度，densityPixels屏幕密度像素=屏幕宽度像素/屏幕真实宽度
  let realWindowWidth = windowWidth / display.getDefaultDisplaySync().densityPixels;
  hilog.info(DOMAIN, 'testTag', `updateScreenDisplay windowWidth:${windowWidth}, windowWidthVp:${realWindowWidth}`);
  let curScreenSizeString: string = '';
  if (realWindowWidth < CommonConstant.SCREEN_SIZE_BREAKPOINT) {
    curScreenSizeString = CommonConstant.SCREEN_SIZE_STRING_SM;
  } else {
    curScreenSizeString = CommonConstant.SCREEN_SIZE_STRING_LG;
  }
  AppStorage.setOrCreate('curScreenSizeString', curScreenSizeString);
}
```

### The breakpoint constants, with the measurements they were derived from

`APPMultiplier.zip#APPMultiplier/entry/src/main/ets/constant/CommonConstant.ets`

```ts
export class CommonConstant {
  // mate60屏幕真实宽度如下：
  // 竖屏：374.15384615384613
  // 横屏：827.0769230769231
  // X6折叠态屏幕真实宽度如下：
  // 竖屏：345.6
  // 横屏：780.8
  // X6展开态屏幕真实宽度如下：
  // 竖屏：716.8
  // 横屏：780.8

  static readonly SCREEN_SIZE_STRING_SM = 'sm';
  static readonly SCREEN_SIZE_STRING_LG = 'lg';
  static readonly SCREEN_SIZE_BREAKPOINT: number = 640;
  static readonly WEB: string = 'web';
  static readonly TAB_BAR_WIDTH_PERCENT: string = '100%';
  static readonly TAB_BAR_WIDTH_PERCENT_LG: string = '10%';
  static readonly TAB_BAR_HEIGHT_PERCENT: string = '10%';
  static readonly TAB_BAR_HEIGHT_PERCENT_LG: string = '100%';
  static readonly DEVICE_PAGE_INDEX: number = 1;
}
```

### Binding the mode

`APPMultiplier.zip#APPMultiplier/entry/src/main/ets/pages/AppMultiplier.ets`

```ts
// curScreenSizeString需初始化，但实际不赋值，用AppStorage.setOrCreate存储的值
@StorageLink('curScreenSizeString') curScreenSizeString: string = CommonConstant.SCREEN_SIZE_STRING_SM;

// ...
.mode(this.curScreenSizeString === CommonConstant.SCREEN_SIZE_STRING_LG ? NavigationMode.Split :
  NavigationMode.Stack)
```

### Loading the list data from a rawfile

`APPMultiplier.zip#APPMultiplier/entry/src/main/ets/toolability/APPMultiplierTools.ets`

```ts
export async function getDataFromJSON(context: Context, fileName: string) {
  let result: DisplayInfo[] = [];
  try {
    let value: Uint8Array = await context.resourceManager.getRawFileContent(fileName);
    let str = buffer.from(value.buffer).toString();
    result = JSON.parse(str) as DisplayInfo[];
  } catch (err) {
    hilog.error(0x0000, 'testTag', `err msg is ${err.message}`);
  }
  return result;
}
```

### The detail destination

`APPMultiplier.zip#APPMultiplier/entry/src/main/ets/toolability/APPMultiplierTools.ets`

```ts
@Builder
export function buildNavDestination(name: string, param: NavPathInfoParamObj) {
  NavDestination() {
    List() {
      ListItem() {
        DisplayDeviceItem({ name: '设备名称', value: param.item.deviceName });
      };
      ListItem() {
        DisplayDeviceItem({ name: '设备颜色', value: param.item.deviceColor });
      };
      ListItem() {
        DisplayDeviceItem({ name: '设备内存', value: param.item.deviceSize });
      };
    }
    .divider({ strokeWidth: 1, startMargin: 20, endMargin: 20 })
    .backgroundColor('#FFFFFF')
    .borderRadius(16)
    .width('90%');
  }
  .title('设备信息')
  .backgroundColor('#F1F3F5');
}
```

## Permissions & config

**No permissions are required** and none are declared - the feature reads only
the application's own window geometry.

`APPMultiplier.zip#APPMultiplier/entry/src/main/module.json5` declares the usual
`EntryAbility` with the home skill and an `EntryBackupAbility`. There is no
`requestPermissions` block.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. `on`/`off('windowSizeChange')`
  date from API 7.
- **`windowSizeChange` reports px.** The breakpoint comparison must convert -
  either by `densityPixels` as here, or with `UIContext.px2vp`.
- **`on('windowSizeChange')` can be called only by the main thread**, and so can
  `off`.
- **The subscription reports changes only.** The initial width has to be read
  separately from `getWindowProperties().windowRect`.
- **640 vp is a custom breakpoint.** The platform's default grid breakpoints are
  `['320vp', '600vp', '840vp']`; the reference explicitly allows custom values -
  "In practice, set breakpoint values based on actual usage scenarios" - and the
  sample's comment block records the folded/unfolded widths that motivated 640.
  Do not assume `'lg'` here means the standard `lg` breakpoint.
- **Landscape blurs the distinction.** By the sample's own measurements an X6 is
  780.8 vp wide in landscape both folded and unfolded, so a width-only rule puts
  the folded device into Split mode when it is rotated.
- **`DisplayDeviceItem` hardcodes `.height('192px')`** - a px literal in an
  otherwise density-independent layout.
- **Devices.** Per `module.json5`; the feature is meaningful on foldables and
  wide screens.

## Pitfalls

- **The `windowSizeChange` subscription is never released, which is incorrect.**
  `onWindowStageDestroy` only logs, and the reference documents
  `off('windowSizeChange')` in both its forms. On a foldable the handler fires on
  every fold, so the leak is not theoretical. (HW-19-0098)
- **`getMainWindow()` has no rejection handler, which is incorrect.** If it
  rejects, no breakpoint is ever published and no listener is installed - the
  adaptation silently does nothing, with no log line. The `loadContent` call four
  lines below does check its error. (HW-19-0099)
- **Width alone is not fold state.** Both branches of the sample's own comment
  block show the X6 at 780.8 vp in landscape whether folded or unfolded, so the
  rule reports `lg` for a folded device held sideways. If the distinction
  matters, read the fold status rather than the width.
- **The initial read and the change handler must agree.**
  `getWindowProperties().windowRect.width` and the `windowSizeChange` payload are
  both px here; mixing one px source with one vp source is a common way to get a
  breakpoint that is right at launch and wrong afterwards.
- **`@StorageLink` initialisers are placeholders.** The comment says so -
  "curScreenSizeString需初始化，但实际不赋值" ("curScreenSizeString must be
  initialised, but is not actually assigned") - the real value always comes from
  the ability.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `on`/`off('windowSizeChange')` with both off forms, `getWindowProperties`, and
  the main-thread restriction.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-windowstage.md` -
  `WindowStage.getMainWindow`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-windowstage
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-basic-components-navigation -
  `Navigation`, `NavigationMode` and the `mode` attribute.
- `documentation/harmonyos-guides/03_application-framework/arkts-layout-development-grid-layout.md` -
  the default breakpoint array `['320vp', '600vp', '840vp']` and the guidance on
  choosing custom breakpoints.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-layout-development-grid-layout
- `documentation/harmonyos-references/02_application-framework/js-apis-display.md` -
  `display.getDefaultDisplaySync()` and `densityPixels`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-display
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-split-mode.md` -
  the Split-mode layout being adapted to.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-split-mode
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_multiplier-0000002312850858
