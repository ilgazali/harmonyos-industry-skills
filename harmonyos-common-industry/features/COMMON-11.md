---
id: COMMON-11
title: App theme edition switch - persist the chosen edition and swap the Navigation root between a standard and a large-text home
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/11_edition_switch.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/edition_switch-0000002284505357
sample: huawei_industry_tree/19_common_technical_solutions/downloads/EditionSwitch.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: ["PersistentStorage.persistProp", "@StorageLink", "@StorageProp", AppStorage, Navigation, NavPathStack, NavDestination, "NavPathStack.pushPathByName", "NavPathStack.pop", "NavDestination.onReady", "NavDestination.onShown", "NavDestination.onHidden", NavDestinationContext, routerMap, Radio, RadioIndicatorType, "window.setWindowSystemBarProperties", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "window.on('avoidAreaChange')", "component .safeAreaPadding", "component .colorFilter", "PromptAction.showToast"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0019, HW-19-0020, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when the product offers **user-selectable theme editions** - a
standard edition and a large-text / accessibility ("关怀版", "老年版") edition, or a
seasonal festival skin - where the two editions differ enough that they are
separate home screens rather than a set of style overrides.

The selection has to survive a restart, and switching has to take effect
immediately without relaunching the app.

## Feature checklist

The application must:

- Define an edition as a numeric mode constant (`STANDARD_MODE = 0`,
  `LARGE_WORD_MODE = 1`).
- Persist the selected mode so the next launch reopens in the same edition.
- **Register the persisted property before anything reads it** - otherwise the
  previous run's value is destroyed (HW-19-0019).
- Bind the mode two-way into the UI so a change re-renders immediately.
- Swap the whole root content of the `Navigation` based on the mode.
- Offer a switch page listing the available editions with a radio selection, a
  confirmation toast and an automatic return to the home page.
- Adapt the system status-bar content colour per edition, and restore it when
  leaving the switch page.
- Lay out full-screen and pad for the status bar and navigation indicator.

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | publishes the `WindowStage` and the two insets into `AppStorage`, sets full-screen layout, subscribes to `avoidAreaChange`, loads `pages/HomePage`, and calls `PersistentStorage.persistProp('mode', ...)` |
| `common/CommonConstants.ets` | `STANDARD_MODE`, `LARGE_WORD_MODE`, and the two `ColorFilter` matrices used to tint tab icons |
| `model/EditionDescription.ets` | `ModeDescription` interface and `MODE_LIST`, the two edition rows shown on the switch page |
| `model/IconInfo.ets` | icon/banner data for the home screens |
| `pages/HomePage.ets` | the only entry in `main_pages.json`; owns the `NavPathStack`, holds `@StorageLink('mode')`, and picks the home component |
| `pages/HomeStandard.ets`, `pages/HomeLargeWord.ets` | the two edition home screens, each exported through a builder registered in `route_map.json` |
| `pages/EditionSwitch.ets` | the `NavDestination` with the radio list |
| `components/HeaderToolBar.ets` | the header that navigates to the switch page with `pushPathByName('EditionSwitch', '')` |
| `components/Banner.ets` | shared banner component |
| `utils/StatusBarUtils.ets` | light/dark status-bar helpers driven by the current mode |
| `utils/Logger.ets` | thin `hilog` wrapper (`LOGGER`) |

**The switching mechanism is a conditional inside `Navigation`, not a route
change.** `HomePage` renders `HomeStandard()` or `HomeLargeWord()` depending on
`this.mode`; because `mode` is `@StorageLink`, writing it anywhere in the app
re-evaluates that `if` and swaps the entire root subtree. The routes in
`route_map.json` (`StandardHome`, `LargeWordHome`, `EditionSwitch`) exist so the
same components can also be pushed as destinations, but the edition swap itself
never touches the stack.

**State flow.**

1. `PersistentStorage.persistProp('mode', STANDARD_MODE)` seeds AppStorage from
   the persisted file (this must happen first - see HW-19-0019).
2. `HomePage` binds `@StorageLink('mode')`, so it reads the seeded value and
   re-renders on any change.
3. `EditionSwitch` binds the same `@StorageLink('mode')`. Selecting a radio
   assigns `this.mode = index`.
4. The assignment propagates to AppStorage, from AppStorage to
   `PersistentStorage` (writing it to disk), and back to `HomePage`, which swaps
   the root component.
5. `EditionSwitch` shows a toast and pops itself off the stack; the user lands on
   the newly themed home.

Note that the mode value **is** the index into `MODE_LIST` - `EditionSwitch`
writes `this.mode = index` directly - so the order of `MODE_LIST` and the values
of the mode constants are one and the same contract.

## Implementation steps

1. **Define the mode constants** and keep `MODE_LIST` in the same order.
2. **Register the persisted property first.** Call
   `PersistentStorage.persistProp('mode', STANDARD_MODE)` **before**
   `windowStage.loadContent(...)` - at the top of `onWindowStageCreate` or in
   `onCreate`. The shipped sample calls it inside the `loadContent` callback,
   which is too late (HW-19-0019).
3. **Publish the window stage and insets** into `AppStorage` so
   `StatusBarUtils` and the pages can reach them, and keep the insets current
   with `'avoidAreaChange'`.
4. **Bind the mode in the entry page** with `@StorageLink('mode')` and render the
   edition conditionally inside `Navigation(this.pageStack)`.
5. **Provide the stack** from the entry page (`@Provide('NavPathStack')`) so
   nested components can push without prop-drilling.
6. **Build the switch page as a `NavDestination`**, capture the real stack in
   `onReady((context) => this.pageStack = context.pathStack)`, and pop after the
   selection.
7. **Drive the status bar from the mode** and restore it on `onHidden` of the
   switch page.
8. **Consider PersistenceV2.** The official guide advises `globalConnect` over
   `persistProp`; decide deliberately (HW-19-0020).

## Verified snippets

All snippets below come from the sample project, not from the document.

### The edition swap

`EditionSwitch.zip#EditionSwitch/entry/src/main/ets/pages/HomePage.ets`

```ts
import { STANDARD_MODE } from '../common/CommonConstants';
import { StatusBarUtils } from '../utils/StatusBarUtils';
import { HomeLargeWord } from './HomeLargeWord';
import { HomeStandard } from './HomeStandard';

@Entry
@Component
struct Index {
  @StorageLink('mode') mode: number = STANDARD_MODE;
  @Provide('NavPathStack') pageStack: NavPathStack = new NavPathStack();

  aboutToAppear(): void {
    StatusBarUtils.changeStatusBarByMode();
  }

  build() {
    Navigation(this.pageStack) {
      if (this.mode === STANDARD_MODE) {
        HomeStandard();
      } else {
        HomeLargeWord();
      }
    }
    .width('100%')
    .height('100%')
    .hideToolBar(true);
  }
}
```

### Registering the persisted property (as shipped - too late, see HW-19-0019)

`EditionSwitch.zip#EditionSwitch/entry/src/main/ets/entryability/EntryAbility.ets`

```ts
AppStorage.setOrCreate('windowStage', windowStage);

let windowClass: window.Window = windowStage.getMainWindowSync();
windowClass.setWindowLayoutFullScreen(true)
  .then(() => {
    let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR;
    AppStorage.setOrCreate('bottomRectHeight', windowClass.getWindowAvoidArea(type).bottomRect.height);
    type = window.AvoidAreaType.TYPE_SYSTEM;
    AppStorage.setOrCreate('topRectHeight', windowClass.getWindowAvoidArea(type).topRect.height);
  })
  .catch((err: BusinessError) => {
    LOGGER.error(`Failed to set full screen. Cause: ${JSON.stringify(err)}`);
  });

windowClass.on('avoidAreaChange', (data) => { /* keeps both heights current */ });

windowStage.loadContent('pages/HomePage', (err) => {
  if (err.code) {
    hilog.error(0x0000, 'testTag', 'Failed to load the content. Cause: %{public}s', JSON.stringify(err) ?? '');
    return;
  }
  PersistentStorage.persistProp('mode', STANDARD_MODE); // FIX (HW-19-0019): move above loadContent
  hilog.info(0x0000, 'testTag', 'Succeeded in loading the content.');
});
```

Corrected ordering:

```ts
onWindowStageCreate(windowStage: window.WindowStage): void {
  PersistentStorage.persistProp('mode', STANDARD_MODE); // first: seeds AppStorage from disk
  AppStorage.setOrCreate('windowStage', windowStage);
  // ... window setup ...
  windowStage.loadContent('pages/HomePage', (err) => { /* ... */ });
}
```

### The switch page

`EditionSwitch.zip#EditionSwitch/entry/src/main/ets/pages/EditionSwitch.ets`

```ts
@Builder
export function EditionSwitchBuilder() {
  EditionSwitch();
}

@Component
struct EditionSwitch {
  @StorageProp('topRectHeight') topRectHeight: number = 0;
  @StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
  @StorageLink('mode') mode: number = STANDARD_MODE;
  private pageStack?: NavPathStack;

  build() {
    NavDestination() {
      List() {
        ForEach(MODE_LIST, (item: ModeDescription, index: number) => {
          ListItem() {
            Row({ space: 12 }) {
              Image(item.icon);
              Column() {
                Text(item.name);
                Text(item.desc);
              }.alignItems(HorizontalAlign.Start);
              Blank().layoutWeight(1);
              Radio({
                value: `radio${index}`,
                group: 'radioGroup',
                indicatorType: RadioIndicatorType.DOT
              })
                .checked(this.mode === index)
                .radioStyle({ checkedBackgroundColor: $r('app.color.tab_selected_color') })
                .onChange((isChecked) => {
                  if (isChecked) {
                    this.mode = index;
                    this.getUIContext().getPromptAction().showToast({ message: $r('app.string.toast_switch_success') });
                    this.pageStack?.pop();
                  }
                });
            }
            .width('100%')
            .onClick(() => {
              this.mode = index;
            });
          };
        }, (item: ModeDescription) => JSON.stringify(item));
      }
      .width('100%')
      .height('100%');
    }
    .safeAreaPadding({
      bottom: this.getUIContext().px2vp(this.bottomRectHeight),
      top: this.getUIContext().px2vp(this.topRectHeight)
    })
    .onReady((context: NavDestinationContext) => {
      this.pageStack = context.pathStack;
    })
    .onShown(() => StatusBarUtils.changeStatusBarToDarkTheme())
    .onHidden(() => StatusBarUtils.changeStatusBarByMode())
    .title($r('app.string.title_version_switch'));
  }
}
```

### Status-bar helpers driven by mode

`EditionSwitch.zip#EditionSwitch/entry/src/main/ets/utils/StatusBarUtils.ets`

```ts
export class StatusBarUtils {
  static windowStage: window.WindowStage = AppStorage.get('windowStage') as window.WindowStage;

  static changeStatusBarToDarkTheme() {
    let mainWin: window.Window = StatusBarUtils.windowStage.getMainWindowSync();
    let sysBarProps: window.SystemBarProperties = {
      statusBarColor: '#00000000',
      statusBarContentColor: '#000000'
    };
    mainWin.setWindowSystemBarProperties(sysBarProps).catch((err: BusinessError) => {
      LOGGER.error('Failed to set the system bar properties. Cause: ' + JSON.stringify(err));
    });
  }

  static changeStatusBarByMode() {
    let mode: number = AppStorage.get('mode') as number;
    switch (mode) {
      case STANDARD_MODE:
        StatusBarUtils.changeStatusBarToLightTheme();
        break;
      case LARGE_WORD_MODE:
        StatusBarUtils.changeStatusBarToDarkTheme();
        break;
    }
  }
}
```

### Edition list model

`EditionSwitch.zip#EditionSwitch/entry/src/main/ets/model/EditionDescription.ets`

```ts
export interface ModeDescription {
  icon: ResourceStr;
  name: ResourceStr;
  desc: ResourceStr;
  color: ResourceColor;
}

export const MODE_LIST: ModeDescription[] = [
  { icon: $r('app.media.ic_standard'),   name: $r('app.string.name_standard'),   desc: $r('app.string.desc_standard'),   color: $r('app.color.standard_mode_color') },
  { icon: $r('app.media.ic_large_word'), name: $r('app.string.name_large_word'), desc: $r('app.string.desc_large_word'), color: $r('app.color.large_word_mode_color') }
];
```

### Route table

`EditionSwitch.zip#EditionSwitch/entry/src/main/resources/base/profile/route_map.json`

```json
{
  "routerMap": [
    { "name": "StandardHome",  "pageSourceFile": "src/main/ets/pages/HomeStandard.ets",  "buildFunction": "HomeStandardBuilder" },
    { "name": "LargeWordHome", "pageSourceFile": "src/main/ets/pages/HomeLargeWord.ets", "buildFunction": "HomeLargeWordBuilder" },
    { "name": "EditionSwitch", "pageSourceFile": "src/main/ets/pages/EditionSwitch.ets", "buildFunction": "EditionSwitchBuilder" }
  ]
}
```

## Permissions & config

**No permissions are required** - the shipped `module.json5` declares no
`requestPermissions` block, and none of the APIs used need one.

`EditionSwitch.zip#EditionSwitch/entry/src/main/module.json5`:

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
    "routerMap": "$profile:route_map",
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

`main_pages.json` lists only `pages/HomePage`; both editions and the switch page
are Navigation destinations.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`persistProp` must precede every access to the key.** The official guide's
  correct sequence is: persistProp initialises PersistentStorage; the value is
  placed into AppStorage; only then does `@StorageLink` bind to it. Reversing the
  order silently loses the previous run's value.
- **PersistentStorage is module-scoped.** "The data storage path of
  PersistentStorage is at the module level ... If multiple modules use the same
  key, the data belongs to the module that first uses PersistentStorage", and the
  path is "determined when the first ability of the application is started". A
  multi-module app that reads `mode` from several HAPs will not behave the way a
  single-module demo does.
- **Official direction is PersistenceV2.** "you are advised to use the
  **globalConnect** API of PersistenceV2 instead of the **persistProp** API of
  PersistentStorage" (HW-19-0020).
- **`StatusBarUtils.windowStage` is a static field initialised at module load.**
  It reads `AppStorage.get('windowStage')` once; the ability must have published
  the stage before the utils module is first imported. In this project that holds
  because the publication happens before `loadContent`, but any import of
  `StatusBarUtils` from an earlier point would capture `undefined`.
- **The mode value is the `MODE_LIST` index.** Reordering `MODE_LIST` silently
  remaps every persisted preference.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **Calling `PersistentStorage.persistProp` inside the `loadContent` callback is
  incorrect.** By then `@StorageLink('mode')` in `HomePage` has already created
  `mode` in AppStorage with the hardcoded default, so persistProp adopts that
  default and overwrites the stored preference - the app always reopens in
  standard mode. Call persistProp before `loadContent`. (HW-19-0019)
- **The document recommends PersistentStorage without the official caveat, which
  is incomplete.** The state-management guide advises `PersistenceV2.globalConnect`
  instead, because PersistentStorage is coupled to AppStorage and misbehaves
  across modules. (HW-19-0020)
- **The row `onClick` and the radio `onChange` do different things.** Tapping the
  row sets `this.mode = index` only; tapping the radio also shows the toast and
  pops the page. If you reuse this page, route both through one handler.
- **Do not confuse the conditional swap with routing.** The edition change is an
  `if` inside `Navigation`'s content, so the navigation stack is untouched; the
  three entries in `route_map.json` are a separate capability.
- **`StatusBarUtils.changeStatusBarByMode()` reads `AppStorage.get('mode')`
  directly**, so it is subject to the same ordering requirement as
  `@StorageLink` - one more reason persistProp has to run first.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-persiststorage.md` -
  the persistProp initialisation sequence, the "Accessing a Property in AppStorage
  Before PersistentStorage" incorrect-use section, the module-level storage path,
  and the recommendation to use `PersistenceV2.globalConnect`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-persiststorage
- `documentation/harmonyos-guides/03_application-framework/arkts-appstorage.md` -
  `@StorageLink` / `@StorageProp` semantics and AppStorage property creation.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-appstorage
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` -
  `Navigation`, `NavDestination`, `NavPathStack` and the `routerMap` profile.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-e.md` -
  `AvoidAreaType.TYPE_SYSTEM` / `TYPE_NAVIGATION_INDICATOR`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-e
- `documentation/harmonyos-references/02_application-framework/js-apis-window.md` -
  `setWindowSystemBarProperties` and `SystemBarProperties`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-window
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/edition_switch-0000002284505357
