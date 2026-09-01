---
id: COMMON-08
title: Feature guide overlay - highlight real components through a dim mask with @ohos/high_light_guide
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/08_feature_guide_page.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/feature_guide_page-0000002271957437
sample: huawei_industry_tree/19_common_technical_solutions/downloads/FeatureGuidePage.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: [HighLightGuideComponent, HighLightGuideBuilder, GuidePage, HighLightOptionsBuilder, HighLightShape, Controller, "GuidePage.addHighLightWithOptions", "GuidePage.setHighLightIndicator", "HighLightGuideBuilder.setLabel", "HighLightGuideBuilder.alwaysShow", "HighLightGuideBuilder.addGuidePage", "component .id()", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "AvoidAreaType.TYPE_SYSTEM", "UIContext.px2vp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0009, HW-19-0010, HW-19-0011, HW-19-0012, HW-19-0181, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when the product needs a **first-launch feature walkthrough**: a
dark mask over the real home page with one live component cut out and highlighted
at a time, plus an arrow, a caption and a "next" button, stepping through several
components in sequence.

The distinguishing property of this approach is that the highlighted region is a
**real component of the real page**, located by its `id` at runtime - not a
screenshot and not a mocked-up copy of the layout. The page underneath is the
production page; the guide is a layer on top of it.

The implementation uses the third-party library
[`@ohos/high_light_guide`](https://ohpm.openharmony.cn/#/cn/detail/@ohos%2Fhigh_light_guide)
(the sample pins `^1.0.4`); there is no system API for this.

## Feature checklist

The application must:

- Render the normal home page as the guide's underlying content, unmodified.
- Give every component that will be highlighted a **unique** `id`.
- Build one guide page per walkthrough step, each binding one component id and one
  overlay layout.
- Highlight with a shape and corner radius that match the highlighted component.
- Re-measure the highlighted component's position when the layout can change
  (`isFetchLocationEveryTime(true)`).
- Show the overlay automatically once the guide component reports ready.
- Provide a per-step overlay layout: arrow, explanatory text, action button, and
  any decorative copy of a bottom-tab item.
- Decide whether the guide shows once per user or on every launch
  (`alwaysShow`).
- Pad the page for the status bar and navigation indicator, since the window is
  laid out full-screen.

## Architecture

Single-module project (`entry` HAP), five source files plus a backup extension:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | loads `pages/MultiGuidePage`, forces light colour mode, sets full-screen layout, publishes the top and bottom insets into `AppStorage` |
| `entrybackupability/EntryBackupAbility.ets` | backup ExtensionAbility declared in `module.json5` |
| `common/Constants.ets` | layout constants plus `COMPONENT_ID`, the array of highlight target ids |
| `model/IconModel.ets`, `model/MusicModel.ets` | plain data classes for the list items |
| `pages/HomePage.ets` | the real music home page - banner, playlist recommendations, new-music swiper, now-playing bar, bottom tabs - and the place where `.id()` is applied |
| `pages/MultiGuidePage.ets` | the guide: builds the `HighLightGuideBuilder`, hosts `HighLightGuideComponent`, and defines the per-step overlay builders |

The layering, and the contract between the two pages, is the whole design:

1. `HomePage` marks its highlightable components with ids from `COMPONENT_ID`.
2. `MultiGuidePage.aboutToAppear` builds a `HighLightGuideBuilder`: one
   `GuidePage.newInstance()` per step, each calling
   `addHighLightWithOptions(id, shape, radius, padding, options)` to name the
   target and `setHighLightIndicator(builder)` to attach that step's overlay.
3. `HighLightGuideComponent` receives `highLightContainer: this.HighLightComponent`
   (a `@Builder` that renders `HomePage()`) and the builder, and calls back
   `onReady(controller)`.
4. `onReady` stores the `Controller` and calls `controller.show()`, which draws the
   mask, cuts out the region of the component whose id matches, and renders the
   step's indicator on top.

The library cuts the highlight with a Canvas rounded rectangle, which is where the
document's one caveat comes from - see Constraints.

## Implementation steps

1. **Add the dependency.** `"@ohos/high_light_guide": "^1.0.4"` in
   `oh-package.json5`, then `ohpm install`.
2. **Declare the highlight ids in one place.** A `COMPONENT_ID` string array in
   `common/Constants.ets` keeps the page and the guide in sync.
3. **Tag the target components in the real page** with `.id(COMPONENT_ID[n])`.
   Tag exactly one component instance per id - not every item of a `ForEach`
   (HW-19-0010) - and keep the index within the array's bounds (HW-19-0009).
4. **Build the guide in `aboutToAppear`.** `new HighLightGuideBuilder()`, then
   `.setLabel(label)` with a **resolved string** (HW-19-0011), `.alwaysShow(true)`
   while developing, then one `.addGuidePage(...)` per step.
5. **Per step**, chain
   `GuidePage.newInstance().addHighLightWithOptions(id, HighLightShape.ROUND_RECTANGLE,
   radius, padding, new HighLightOptionsBuilder().isFetchLocationEveryTime(true).build())
   .setHighLightIndicator(this.stepIndicator)`. Match `radius` to the target's own
   `borderRadius`.
6. **Write the underlying page as a `@Builder`** that simply renders the real page
   component, and pass it as `highLightContainer`.
7. **Mount `HighLightGuideComponent`** with `builder`, `currentHLIndicator: null`
   and an `onReady` callback that stores the controller and calls `show()`.
8. **Write one `@Builder` per step** for the overlay: arrow image, caption text,
   next button, and any decorative element. Position them with `.position({x, y})`
   in percentages so they follow the highlight across screen sizes.
9. **Handle the window insets.** Set full-screen layout in
   `onWindowStageCreate`, read the **`TYPE_SYSTEM`** avoid area for the status bar
   and `TYPE_NAVIGATION_INDICATOR` for the bottom bar (HW-19-0012), publish both
   into `AppStorage`, and consume them with `@StorageProp` + `px2vp` as page
   padding.

## Verified snippets

All snippets below come from the sample project, not from the document.

### Highlight target ids

`FeatureGuidePage.zip#FeatureGuidePage/entry/src/main/ets/common/Constants.ets`

```ts
export const COMPONENT_ID: string[] = ['musicFM', 'playlistRecommend', 'newMusicRecommend'];
```

### Building the multi-step guide

`FeatureGuidePage.zip#FeatureGuidePage/entry/src/main/ets/pages/MultiGuidePage.ets`

```ts
import Constants, { COMPONENT_ID } from '../common/Constants';
import { HomePage } from './HomePage';
import {
  HighLightGuideComponent,
  HighLightGuideBuilder,
  Controller,
  GuidePage,
  HighLightOptionsBuilder,
  HighLightShape,
} from '@ohos/high_light_guide';

@Entry
@Component
struct MultiGuidePage {
  @State labelText: ResourceStr = $r('app.string.high_light_label');
  private builder: HighLightGuideBuilder | null = null;
  private controller: Controller | null = null;

  aboutToAppear(): void {
    this.builder = new HighLightGuideBuilder()
      .setLabel(this.labelText as string) // FIX (HW-19-0011): resolve the resource to a string first
      .alwaysShow(true)
      .addGuidePage(GuidePage.newInstance()
        .addHighLightWithOptions(COMPONENT_ID[0], HighLightShape.ROUND_RECTANGLE, Constants.HIGH_LIGHT_ROUND0,
          Constants.HIGH_LIGHT_PADDING, new HighLightOptionsBuilder().isFetchLocationEveryTime(true).build())
        .setHighLightIndicator(this.firstIndicator)
      )
      .addGuidePage(GuidePage.newInstance()
        .addHighLightWithOptions(COMPONENT_ID[1], HighLightShape.ROUND_RECTANGLE, Constants.HIGH_LIGHT_ROUND1,
          Constants.HIGH_LIGHT_PADDING, new HighLightOptionsBuilder().isFetchLocationEveryTime(true).build())
        .setHighLightIndicator(this.secondIndicator)
      );
  }

  build() {
    Column() {
      Stack() {
        HighLightGuideComponent({
          highLightContainer: this.HighLightComponent,
          currentHLIndicator: null,
          builder: this.builder,
          onReady: (controllerParam: Controller) => {
            this.controller = controllerParam;
            this.controller.show();
          }
        })
          .width(Constants.FULL_PERCENT)
          .height(Constants.FULL_PERCENT);
      };
    }
    .width(Constants.FULL_PERCENT);
  }

  @Builder
  private HighLightComponent(): void {
    Column() {
      HomePage();
    }.alignItems(HorizontalAlign.Center);
  }
}
```

### A step overlay

`FeatureGuidePage.zip#FeatureGuidePage/entry/src/main/ets/pages/MultiGuidePage.ets`

```ts
@Builder
private firstIndicator(): void {
  Column() {
    Column() {
      Image($r('app.media.arrow'))
        .height($r('app.float.height_48'))
        .margin({ left: $r('app.float.margin_90') });
      Text($r('app.string.first_indicator_text'))
        .fontSize($r('app.float.font_size_14'))
        .fontWeight(FontWeight.Regular)
        .textAlign(TextAlign.Center)
        .fontColor(Color.White)
        .width(Constants.PERCENT_60)
        .margin($r('app.float.margin_8'));
    }.alignItems(HorizontalAlign.Start);

    Button($r('app.string.button_text'))
      .backgroundColor($r('app.color.button_background_color'))
      .fontColor($r('app.color.start_window_background'))
      .fontSize($r('app.float.font_size_14'))
      .margin($r('app.float.margin_8'))
      .width($r('app.float.width_120'));
  }.width(Constants.FULL_PERCENT)
  .position({ x: Constants.NEGATIVE_PERCENT_15, y: Constants.PERCENT_52 })
  .alignItems(HorizontalAlign.Center);
}
```

### Tagging the target component in the real page

`FeatureGuidePage.zip#FeatureGuidePage/entry/src/main/ets/pages/HomePage.ets`

```ts
// as shipped - the id is applied to every ForEach item, see HW-19-0010
ForEach(this.topBannerList, (bannerItem: ResourceStr, index: number) => {
  ListItem() {
    Image(bannerItem)
      .width($r('app.float.width_224'))
      .height($r('app.float.height_280'))
      .borderRadius($r('app.float.radius_16'))
      .id(COMPONENT_ID[0])
      .margin({ left: index === Constants.BANNER_INDEX0 ? $r('app.float.margin_16') : $r('app.float.margin_8') });
  };
});

// corrected form - exactly one component carries the id
ForEach(this.topBannerList, (bannerItem: ResourceStr, index: number) => {
  ListItem() {
    Image(bannerItem)
      .borderRadius($r('app.float.radius_16'))
      .id(index === Constants.BANNER_INDEX0 ? COMPONENT_ID[0] : '');
  };
});
```

### Window insets (corrected - see HW-19-0012)

`FeatureGuidePage.zip#FeatureGuidePage/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
// as shipped
let avoidArea = windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR);
AppStorage.setOrCreate('bottomRectHeight', avoidArea.bottomRect.height);
let cutoutArea = windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_CUTOUT);
AppStorage.setOrCreate('topRectHeight', cutoutArea.topRect.height + 40);

// corrected: TYPE_SYSTEM is the status bar area
let systemArea = windowClass.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM);
AppStorage.setOrCreate('topRectHeight', systemArea.topRect.height);
```

Consumed in `HomePage.ets`:

```ts
@StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
@StorageProp('topRectHeight') topRectHeight: number = 0;
// ...
.padding({
  top: this.getUIContext().px2vp(this.topRectHeight),
  bottom: this.getUIContext().px2vp(this.bottomRectHeight)
})
```

## Permissions & config

**No permissions are required.** The feature is pure UI; neither the document nor
the sample declares any `requestPermissions` entry.

`FeatureGuidePage.zip#FeatureGuidePage/oh-package.json5` - the only dependency:

```json5
{
  "modelVersion": "6.0.0",
  "dependencies": {
    "@ohos/high_light_guide": "^1.0.4"
  }
}
```

`FeatureGuidePage.zip#FeatureGuidePage/entry/src/main/module.json5`:

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
    ],
    "extensionAbilities": [
      {
        "name": "EntryBackupAbility",
        "srcEntry": "./ets/entrybackupability/EntryBackupAbility.ets",
        "type": "backup",
        "exported": false,
        "metadata": [
          { "name": "ohos.extension.backup", "resource": "$profile:backup_config" }
        ]
      }
    ]
  }
}
```

`FeatureGuidePage.zip#FeatureGuidePage/build-profile.json5` pins both
`targetSdkVersion` and `compatibleSdkVersion` to `6.0.0(20)`, with `strictMode`
`caseSensitiveCheck` and `useNormalizedOHMUrl` enabled.

## Constraints

- **API level.** The document requires API Version 20 Release or later, HarmonyOS
  6.0.0 Release SDK or later, and DevEco Studio 6.0.0 Release or later; the
  project pins `6.0.0(20)`.
- **Third-party dependency.** `@ohos/high_light_guide` v1.0.4 from the OpenHarmony
  package repository. This is not a system capability, so the guide's behaviour is
  bounded by that library's version.
- **Known rounded-corner artefact - stated by the document as current
  specification.** "由于Canvas和Image两种圆角绘制方式存在差异，故功能引导中圆角可能会
  存在白边问题，当前规格如此。" ("Because the two rounded-corner drawing methods -
  Canvas and Image - differ, a white edge may appear on the rounded corners in the
  feature guide; this is the current specification.") The library draws the
  highlight cut-out with the Canvas `roundRect` interface, which calls drawing's
  `AddRoundRect`, while `Image.borderRadius` uses the system default corner, "一般
  为G2圆角，非标准圆弧" ("usually a G2 corner, not a standard circular arc"). Expect
  a hairline white edge; do not treat it as a bug in your integration.
- **`id` uniqueness is the developer's responsibility.** The official reference:
  "Sets a unique identifier for this component, with uniqueness guaranteed by the
  user." The library has no way to disambiguate duplicates.
- **Devices.** `phone`, `tablet`, `2in1` per `module.json5`.
- **`alwaysShow(true)` is a demo setting.** In production the guide should be
  suppressed after the user has seen it.

## Pitfalls

- **`.id(COMPONENT_ID[3])` on the music item is incorrect - the array has three
  elements.** The expression is `undefined`, so that highlight target does not
  exist; use `COMPONENT_ID[2]`, the declared `'newMusicRecommend'` entry, which is
  otherwise dead. (HW-19-0009)
- **Applying the highlight id inside `ForEach` is incorrect.** Two banner images
  share `COMPONENT_ID[0]` and three recommendation images share `COMPONENT_ID[1]`,
  breaking the documented uniqueness contract of `id` and making the highlighted
  item depend on lookup order. Tag one instance, or tag the enclosing `List`.
  (HW-19-0010)
- **`.setLabel(this.labelText as string)` is incorrect.** `$r()` yields a
  `Resource` object and `as string` is a compile-time assertion, not a
  conversion; resolve with `resourceManager.getStringSync($r(...).id)` or use a
  plain string. (HW-19-0011)
- **Reading the status-bar height from `TYPE_CUTOUT` and adding 40 is
  incorrect.** `TYPE_SYSTEM` is documented as the status bar area and
  `TYPE_CUTOUT` as the cutout area; on a device with no cutout the shipped code
  pads by a fixed 40 px. Use `TYPE_SYSTEM` with no additive constant.
  (HW-19-0012)
- **The insets are read once and never refreshed.** Unlike the CacheClean sample
  (COMMON-07), this project does not subscribe to `'avoidAreaChange'`, so the
  padding is stale after a rotation or a foldable state change. Add the
  subscription if the app supports those.
- **`isFetchLocationEveryTime(true)` matters here.** The highlighted components
  live inside horizontally scrollable `List`s; without re-fetching, the cut-out
  would be drawn at a stale position.

## References

- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-component-id.md` -
  the `id` attribute: "identifies a component uniquely within an application",
  "uniqueness guaranteed by the user".
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-component-id
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-e.md` -
  the `AvoidAreaType` enumeration: `TYPE_SYSTEM` = status bar area,
  `TYPE_CUTOUT` = cutout area, `TYPE_NAVIGATION_INDICATOR` = bottom navigation
  bar.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-e
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-common-method.md` -
  the Canvas `roundRect` interface named by the document's rounded-corner note.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-components-canvas-common-method#roundrect20
- https://ohpm.openharmony.cn/#/cn/detail/@ohos%2Fhigh_light_guide - the
  `@ohos/high_light_guide` library (V1.0.4), the document's only reference.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/feature_guide_page-0000002271957437
