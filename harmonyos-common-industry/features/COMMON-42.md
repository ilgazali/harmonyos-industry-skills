---
id: COMMON-42
title: Dark and light mode - configuring application resources per colour mode
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/42_dark_mode.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/dark_mode-0000002403773657
sample: huawei_industry_tree/19_common_technical_solutions/downloads/DarkMode.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.ArkTS", "@kit.LocalizationKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: ["ApplicationContext.setColorMode", "ConfigurationConstant.ColorMode", "UIAbility.onConfigurationUpdate", Configuration, "$r", "$rawfile", "resourceManager.getRawFileContentSync", "util.TextDecoder.decodeToString", "window.Window.setWindowLayoutFullScreen", "window.Window.getWindowAvoidArea", "window.Window.on('avoidAreaChange')", AppStorage, "@StorageProp", Tabs, TabContent, WaterFlow]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0127, HW-19-0128, HW-19-0129, HW-19-0130, HW-19-0131, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when the application must render correctly in both light and dark
mode - which on HarmonyOS is not an opt-in feature but the default expectation,
since the system switches colour mode underneath the application whether or not
it is prepared.

The mechanism is almost entirely declarative: put dark values in a `dark`
qualifier directory next to `base`, reference everything through `$r()`, and the
resource manager picks the right set. There is one API call
(`setColorMode`) and no rebuild logic. The work is in the discipline - every
colour and every mode-sensitive image has to be a resource, or it will not
switch.

## Feature checklist

- Call `setColorMode(COLOR_MODE_NOT_SET)` once so the application follows the
  system rather than pinning a mode.
- Create `resources/dark/element/color.json` with **the same names** as
  `base/element/color.json`.
- Create `resources/dark/media/` for images that need a dark variant, with
  **identical file names** to their light counterparts.
- Reference every colour as `$r('app.color.*')` and every mode-sensitive image as
  `$r('app.media.*')` - **never** `$rawfile`, which is excluded from qualifier
  matching (HW-19-0128).
- Only override what actually differs; unmatched names fall back to `base`.

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | `setColorMode`, immersive window setup, avoid-area publishing, `onConfigurationUpdate` |
| `pages/MainPage.ets` | the three-tab shell; all colours via `$r('app.color.*')` |
| `components/AllBusiness.ets` | 4-lane service grid |
| `components/ProductList.ets` | `WaterFlow` product grid, data read from rawfile |
| `utils/FileReaderUtils.ets` | reads and decodes `product_detail.json` |
| `utils/Logger.ets` | thin hilog wrapper |
| `resources/base/element/color.json` | seven light colours |
| `resources/dark/element/color.json` | **the same seven names**, dark values |
| `resources/dark/media/` | four images with dark variants |
| `resources/rawfile/` | product JSON and four product images |

**The colour table is the whole design.** Seven names, defined twice:

| name | base | dark |
| --- | --- | --- |
| `start_window_background` | `#FFFFFF` | `#000000` |
| `tab_bar` | `#F5F7F8` | `#33FFFFFF` |
| `text_color` | `#E6000000` | `#FFFFFFFF` |
| `business_text_color` | `#E6000000` | `#E6FFFFFF` |
| `background` | `#FFF1F3F5` | `#FF000000` |
| `product_background` | `#FFFFFFFF` | `#33FFFFFF` |
| `introduction_text_color` | `#66000000` | `#66FFFFFF` |

Note the pattern in the dark column: surfaces are translucent white over black
(`#33FFFFFF`) rather than opaque grays, and text alpha is preserved while the
base colour flips. That is what keeps the depth hierarchy readable when the
background goes to `#FF000000`.

**Only four of seventeen images have dark variants.** `base/media` holds
seventeen files; `dark/media` holds `personal_center.png`,
`shopping_cart.png`, `top_picks.svg`, `wallet.svg`. Everything else resolves from
`base`, because the system searches the matching qualifier directory first and
falls back to `base` when the name is not found there. Overriding only what
differs is the intended usage, not an omission.

**`$rawfile` is outside all of this.** The product images are loaded as
`$rawfile('product1.png')` and never adapt - the guide is explicit that "rawfile
and resfile are raw file directories, which will not match resources based on the
device status." For product photography that is usually correct; the point is
that the document never says so (HW-19-0128).

**`setColorMode(COLOR_MODE_NOT_SET)` means follow the system.** The other two
values pin the application to light or dark regardless of the device. The sample
calls it in `onCreate`, before any UI exists.

**`onConfigurationUpdate` is where a running application learns of a switch.** The
sample publishes `newConfig.colorMode` into `AppStorage` so pages can react - and
then opens an empty dark branch that does nothing (HW-19-0130).

## Implementation steps

1. **Follow the system.** In `EntryAbility.onCreate`:
   `this.context.getApplicationContext().setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_NOT_SET);`
2. **Define the light palette** in `resources/base/element/color.json` with
   semantic names (`background`, `text_color`, `product_background`) rather than
   visual ones.
3. **Create `resources/dark/element/color.json`** with the same names and the
   dark values. Prefer translucent white over black for raised surfaces.
4. **Add `resources/dark/media/`** only for the images that must change, using
   identical file names. Anything not overridden falls back to `base`.
5. **Reference resources everywhere**: `$r('app.color.text_color')`,
   `$r('app.media.wallet')`. A literal `'#F34D4F'` in the code does not switch -
   the sample keeps its price labels that way deliberately, since the accent
   colour reads on both backgrounds.
6. **React at runtime if needed** by overriding `onConfigurationUpdate` and
   publishing `newConfig.colorMode`. Import `Configuration` from
   `@kit.AbilityKit` (HW-19-0127).

## Verified snippets

All snippets below come from the sample project, not from the document.

### Following the system colour mode

`DarkMode.zip#DarkMode/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
import { ConfigurationConstant, UIAbility } from '@kit.AbilityKit';
import { hilog } from '@kit.PerformanceAnalysisKit';
import { window } from '@kit.ArkUI';
import { BusinessError } from '@kit.BasicServicesKit';
import { Configuration } from '@ohos.app.ability.Configuration';   // FIX (HW-19-0127): '@kit.AbilityKit'

const DOMAIN = 0x0000;

export default class EntryAbility extends UIAbility {
  onCreate(): void {
    this.context.getApplicationContext().setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_NOT_SET);
    hilog.info(DOMAIN, 'testTag', '%{public}s', 'Ability onCreate');
  }

  onConfigurationUpdate(newConfig: Configuration): void {
    AppStorage.setOrCreate('colorMode', newConfig.colorMode);
    if (newConfig.colorMode === ConfigurationConstant.ColorMode.COLOR_MODE_DARK) {
      // Set the status bar to dark mode          <- FIX (HW-19-0130): empty branch
    }
  }
};
```

### The avoid-area subscription (registered, never released)

`DarkMode.zip#DarkMode/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
// 2. Obtain the area for layout avoidance to prevent obstruction
let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR;
let avoidArea = windowClass.getWindowAvoidArea(type);
let bottomRectHeight = avoidArea.bottomRect.height;
AppStorage.setOrCreate('bottomRectHeight', bottomRectHeight);
type = window.AvoidAreaType.TYPE_SYSTEM;
avoidArea = windowClass.getWindowAvoidArea(type);
let topRectHeight = avoidArea.topRect.height;
AppStorage.setOrCreate('topRectHeight', topRectHeight);

// 3. Register the listening function to dynamically obtain the avoidance area data.
windowClass.on('avoidAreaChange', (data) => {     // FIX (HW-19-0129): no matching off()
  if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
    let topRectHeight = data.area.topRect.height;
    AppStorage.setOrCreate('topRectHeight', topRectHeight);
  } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
    let bottomRectHeight = data.area.bottomRect.height;
    AppStorage.setOrCreate('bottomRectHeight', bottomRectHeight);
  }
});
```

This sample uses `AvoidAreaType.TYPE_SYSTEM` for the status-bar inset - the
correct constant, and worth noting because several other samples in this industry
use `TYPE_CUTOUT` with a magic number instead.

### Every colour through the resource system

`DarkMode.zip#DarkMode/entry/src/main/ets/pages/MainPage.ets`

```ts
@Entry
@Component
struct MainPage {
  @StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
  @StorageProp('topRectHeight') topRectHeight: number = 0;
  private controller: TabsController = new TabsController();

  build() {
    Tabs({ barPosition: BarPosition.End, controller: this.controller }) {
      TabContent() {
        Column({ space: 18 }) {
          Row() {
            Text($r('app.string.shopping'))
              .textAlign(TextAlign.Start)
              .fontSize(26)
              .fontWeight(700)
              .margin({ top: 20 })
              .fontColor($r('app.color.text_color')); // Text adaptive color, Avoid setting up unless necessary.
          }
          .width('100%')
          .alignItems(VerticalAlign.Top);

          AllBusiness();

          ProductList();
        }
        .width('100%')
        .height('100%')
        .backgroundColor($r('app.color.background'))
        .padding({
          top: this.getUIContext().px2vp(this.topRectHeight),
          left: 16,
          right: 16
        });
      }.tabBar(
        tabBar($r('app.media.products'), $r('app.string.products'))
      );
      // ... two more TabContent blocks ...
    }
    .vertical(false)
    .scrollable(false)
    .barMode(BarMode.Fixed)
    .fadingEdge(false)
    .backgroundColor($r('app.color.tab_bar')) // Set the background color of the navigation bar
    .barWidth('100%')
    .onContentWillChange(() => {
      return false;
    });
  }
}
```

The inline comment on the `fontColor` line is the sample's own advice: text
inherits a mode-appropriate colour from the framework, so setting it explicitly
is only needed when the design demands a specific one - and then it must be a
resource.

### The tab-bar builder, which sets no colours at all

`DarkMode.zip#DarkMode/entry/src/main/ets/pages/MainPage.ets`

```ts
@Builder
function tabBar(image: Resource, text: Resource) {
  Column() {
    Image(image)
      .width('24')
      .padding({ top: 5 });
    Text(text)
      .fontSize('10')
      .height('13');
  }
  .height(56);
}
```

No `fontColor`, no `fillColor` - the icon comes from `$r('app.media.*')` which
resolves to the `dark/media` variant when one exists, and the label takes the
framework's default. This is the cheapest correct way to write a mode-aware
component.

### The two card surfaces that carry the dark palette

`DarkMode.zip#DarkMode/entry/src/main/ets/components/ProductList.ets`

```ts
@Builder
listItem(info: ProductDetail) {
  Column() {
    Image($rawfile(`${info.image}.png`))     // FIX (HW-19-0128): rawfile never adapts
      .width('100%')
      .height(195)
      .borderRadius({ topLeft: 16, topRight: 16 });
    Column({ space: 4 }) {
      Text(info.name)
        .fontSize(14)
        .fontWeight(500)
        .fontColor($r('app.color.text_color'));
      Text(info.description)
        .fontSize(12)
        .fontWeight(400)
        .fontColor($r('app.color.introduction_text_color'));
    }
    .width(140)
    .height(38)
    .margin({ top: 8 })
    .alignItems(HorizontalAlign.Start);

    Text(info.price)
      .fontSize(18)
      .fontWeight(700)
      .fontColor('#F34D4F')
      .width(140)
      .height(24)
      .margin({ top: 12 });
  }
  .width('100%')
  .height('100%')
  .backgroundColor($r('app.color.product_background'))
  .borderRadius(16);
}
```

`product_background` is `#FFFFFFFF` in light and `#33FFFFFF` in dark - an opaque
white card on a gray page, and a 20%-white card on a black page.

### Reading the product data

`DarkMode.zip#DarkMode/entry/src/main/ets/utils/FileReaderUtils.ets`

```ts
export class FileReader {
  static readRawfile(resourceManager: resourceManager.ResourceManager): ProductDetail[] {
    try {
      let dataStr = FileReader.uint8ArrayToString(resourceManager.getRawFileContentSync('product_detail.json'));
      let dayData = JSON.parse(dataStr) as ProductDetail[];
      return dayData;
    } catch (err) {
      // FIX (HW-19-0131): names word_list.json, reads product_detail.json
      logger.error(`Failed to read word data from rawfile word_list.json. Code: ${err.code}, message: ${err.message}`);
      return [];
    }
  }

  static uint8ArrayToString(array: Uint8Array) {
    let textDecoderOptions: util.TextDecoderOptions = {
      fatal: false,
      ignoreBOM: true
    };
    let decodeToStringOptions: util.DecodeToStringOptions = {
      stream: false
    };
    let textDecoder = util.TextDecoder.create('utf-8', textDecoderOptions);
    let retStr = textDecoder.decodeToString(array, decodeToStringOptions);
    return retStr;
  }
}
```

Called from `ProductList.aboutToAppear` with
`this.getUIContext().getHostContext()?.resourceManager`. Note that
`productList` is a plain `private` field, not `@State` - it is assigned before
the first `build()`, so the initial render is correct, but any later reassignment
would not reach the UI.

### The dark colour set

`DarkMode.zip#DarkMode/entry/src/main/resources/dark/element/color.json`

```json
{
  "color": [
    { "name": "start_window_background", "value": "#000000" },
    { "name": "tab_bar", "value": "#33FFFFFF" },
    { "name": "text_color", "value": "#FFFFFFFF" },
    { "name": "business_text_color", "value": "#E6FFFFFF" },
    { "name": "background", "value": "#FF000000" },
    { "name": "product_background", "value": "#33FFFFFF" },
    { "name": "introduction_text_color", "value": "#66FFFFFF" }
  ]
}
```

## Permissions & config

**No permissions are required** and none are declared. Colour-mode adaptation and
rawfile reads need none.

`DarkMode.zip#DarkMode/build-profile.json5`: `targetSdkVersion` and
`compatibleSdkVersion` `6.0.0(20)`.

Resource layout as shipped:

```
entry/src/main/resources/
├── base/{element,media,profile}/
├── dark/{element,media}/
└── rawfile/            <- not listed in the document's 工程目录 (HW-19-0128)
```

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later. `setColorMode` is API 11+.
- **The qualifier directory must be named `dark`** and sits beside `base` under
  `resources`, not inside it.
- **Names must match exactly** between `base` and `dark`. A name present only in
  `dark` is unreachable in light mode; a name present only in `base` is shared by
  both.
- **`rawfile` and `resfile` never participate in qualifier matching** - "rawfile
  and resfile are raw file directories, which will not match resources based on
  the device status."
- **Hard-coded colour literals do not switch.** Neither do colours computed in
  code.
- **`COLOR_MODE_NOT_SET` follows the system;** `COLOR_MODE_LIGHT` and
  `COLOR_MODE_DARK` pin the application and ignore the device setting.
- **Devices.** phone, tablet, 2in1 per `module.json5`.

## Pitfalls

- **`Configuration` is imported from `@ohos.app.ability.Configuration`, which is
  incorrect;** the reference specifies `import { Configuration } from
  '@kit.AbilityKit';`, the same kit two other symbols in that file already come
  from. (HW-19-0127)
- **The document teaches `dark/media` while its own sample loads product images
  through `$rawfile`, which is incorrect as taught** - rawfile is documented never
  to match on device state, and the `rawfile` directory is missing from the
  project tree entirely. Use `$r('app.media.*')` for anything that must change.
  (HW-19-0128)
- **`on('avoidAreaChange')` is registered with no matching `off()`, which is
  incorrect** - `onWindowStageDestroy` says it releases UI resources and releases
  none. Same defect as COMMON-08, COMMON-22 and COMMON-31. (HW-19-0129)
- **`this.format.toUpperCase()` in the Logger constructor does nothing, and the
  dark branch of `onConfigurationUpdate` is empty** - both read as behaviour and
  produce none. (HW-19-0130)
- **The rawfile error message names `word_list.json` while the code opens
  `product_detail.json`, which is incorrect** - the only diagnostic for an empty
  product grid points at a file that does not exist. (HW-19-0131)
- **Do not override every colour in `dark`.** Fall-back to `base` is the
  mechanism, not a gap; duplicating unchanged values doubles the maintenance
  surface for no benefit.
- **Do not pin the mode to make testing easier and then ship it.**
  `COLOR_MODE_LIGHT` in `onCreate` silently disables the entire feature.

## References

- `documentation/harmonyos-guides/01_getting-started/resource-categories-and-access.md` -
  qualifier directories, the `base` fall-back rule, and the statement that
  rawfile/resfile do not match on device state.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/resource-categories-and-access
- `documentation/harmonyos-references/02_application-framework/js-apis-app-ability-configuration.md` -
  `Configuration`, its `colorMode` field, and the required kit import.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-app-ability-configuration
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-inner-application-applicationcontext -
  `ApplicationContext.setColorMode`.
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/ui-dark-light-color-adaptation -
  the colour-mode adaptation guide the document links.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/dark_mode-0000002403773657
