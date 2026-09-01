---
id: TRANS-06
title: Pin an in-app shortcut to the home screen
industry: 06_public_transport
doc: huawei_industry_tree/06_public_transport/docs/06_add_shortcut_to_desktop.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_shortcut_to_desktop-0000002368913597
sample: huawei_industry_tree/06_public_transport/downloads/AddShortcutToDesktop.zip
kits: ["@kit.StoreKit", "@kit.AbilityKit", "@kit.ArkUI"]
apis: [productViewManager.checkPinShortcutPermitted, productViewManager.requestNewPinShortcut, productViewManager.CheckShortcutResult, shortcuts_config.json, Want.parameters, "@StorageLink", "@Watch", NavPathStack.pushPathByName, TapGesture, Matrix4]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-06-0035, HW-06-0036, HW-06-0037, HW-06-0038, HW-06-0051]
status: verified-with-fixes
---

## When to use

Load this card when the app should offer **"add this to my home screen"** from
inside the app - a metro map, a favourite route, a saved ticket, a specific
dashboard. The user gets an icon on the launcher that opens the app straight at
that screen.

This is a different mechanism from `TRANS-03`'s widget and they are easy to
confuse:

| | `TRANS-03` (widget) | `TRANS-06` (shortcut) |
|---|---|---|
| What lands on the home screen | A rendered tile | An app icon |
| Declared in | `form_config.json` + `FormExtensionAbility` | `shortcuts_config.json` + `metadata` |
| Added by | The user, from the icon's long-press menu | The app, in code, on the user's request |
| Can show data | Yes (if dynamic) | No |
| Carries a routing parameter | Yes, via `FormLink` params | Yes, via the want's `parameters` |

Pick the shortcut when you only need a deep link; pick the widget when the tile
itself must show something.

## Feature checklist

- An "add to home screen" control inside the app.
- Adding is checked first, then performed - two calls, not one.
- Trying to add a shortcut that already exists tells the user so.
- The launcher icon carries its own label and icon, distinct from the app's.
- Tapping the shortcut opens the app **at the target screen**.

## Architecture

Single `entry` module.

```
entry/src/main/ets
├── components/ViewImage.ets     the pinch/pan metro map image
├── constants/ImageConstants.ets
├── entryability/EntryAbility.ets catches the shortcut parameter
├── model/TransitInfo.ets
├── pages/MainPage.ets           the add control, and the routing handler
├── pages/MapPage.ets            the target screen
└── utils/Matrix4Util.ets        matrix helpers for the zoomable map
entry/src/main/resources/base/profile/shortcuts_config.json
```

The chain, and it is worth tracing because the sample builds all of it and then
drops the last step:

```
shortcuts_config.json  ──declares──▶  want.parameters.shortCutKey
        │
        ▼ user taps the launcher icon
EntryAbility.onCreate / onNewWant  ──▶  AppStorage['shortCutKey']
        │
        ▼ @StorageLink + @Watch
MainPage.jumpToMap()  ──▶  should push MapPage   ◀── HW-06-0035: it does not
```

`onCreate` **and** `onNewWant` both need the handler: the first covers a cold
start from the shortcut, the second covers tapping it while the app is already
running. The sample gets that part right.

## Implementation steps

1. **Declare the shortcut** in `resources/base/profile/shortcuts_config.json`
   with a `shortcutId`, a label and icon resource, and a `wants` entry carrying
   your routing parameter.
2. **Point `module.json5` at it** from the ability's `metadata`, with the fixed
   name `ohos.ability.shortcuts`.
3. **In code, build a `Want` that matches the profile exactly** - bundle,
   module, ability and parameters (`HW-06-0036`).
4. **Call `checkPinShortcutPermitted` first**, then pass the `tid` it returns to
   `requestNewPinShortcut`. The two-call shape is required: the check both
   validates and issues the token.
5. **Handle failure visibly**, not just in the log (`HW-06-0037`).
6. **Catch the parameter in both `onCreate` and `onNewWant`** and publish it.
7. **Act on it** - navigate (`HW-06-0035`).

## Verified snippets

All snippets are from `AddShortcutToDesktop.zip`. Corrected forms are marked.

**Shortcut declaration — `resources/base/profile/shortcuts_config.json`** (as shipped)

```json
{
  "shortcuts": [
    {
      "shortcutId": "id_map",
      "label": "$string:go_to_map",
      "icon": "$media:metro",
      "wants": [
        {
          "bundleName": "com.example.addshortcuttodesktop",
          "moduleName": "entry",
          "abilityName": "EntryAbility",
          "parameters": {
            "shortCutKey": "MainPage"
          }
        }
      ]
    }
  ]
}
```

**Binding it in `module.json5`** — inside the ability's `metadata`:

```json5
"metadata": [
  {
    "name": "ohos.ability.shortcuts",   // fixed name, do not change
    "resource": "$profile:shortcuts_config"
  }
]
```

**Adding the shortcut — `pages/MainPage.ets`** (corrected, see `HW-06-0036`, `HW-06-0037`)

```typescript
import { productViewManager } from '@kit.StoreKit';

.onClick(() => {
  try {
    // FIX: shipped code names this uiContext, though it is a UIAbilityContext
    const abilityContext = this.getUIContext().getHostContext() as common.UIAbilityContext;
    const shortcutId = 'id_map';        // must match shortcuts_config.json
    const labelResName = 'go_to_map';   // resource *name*, not $r(...)
    const iconResName = 'metro';
    const want: Want = {
      // FIX: shipped code hardcodes 'com.example.addshortcuttodesktop'
      bundleName: abilityContext.abilityInfo.bundleName,
      moduleName: 'entry',
      abilityName: 'EntryAbility',
      parameters: { shortCutKey: 'MainPage' }
    };

    productViewManager.checkPinShortcutPermitted(
      abilityContext, shortcutId, want, labelResName, iconResName)
      .then((result: productViewManager.CheckShortcutResult) => {
        productViewManager.requestNewPinShortcut(abilityContext, result.tid)
          .then(() => {
            Logger.info('requestNewPinShortcut success.');
          })
          .catch((error: BusinessError) => {
            this.toast($r('app.string.add_failed'));   // FIX: shipped code only logs
            Logger.error(`requestNewPinShortcut error. code is ${error.code}`);
          });
      })
      .catch((error: BusinessError) => {
        if (error.code === 1006620003) {
          this.toast($r('app.string.add_already'));    // already on the home screen
        } else {
          this.toast($r('app.string.add_failed'));     // FIX: shipped code only logs
        }
        Logger.error(`checkPinShortcutPermitted error. code is ${error.code}`);
      });
  } catch (err) {
    Logger.error(`add shortcut failed, code is ${err.code}`);
  }
})
```

Three details worth keeping:

- **`labelResName` and `iconResName` are resource *names* as plain strings**,
  not `$r()` references. They index the same resources the profile names.
- **The two-call sequence is mandatory.** `checkPinShortcutPermitted` returns a
  `CheckShortcutResult` whose `tid` is the token `requestNewPinShortcut`
  consumes; there is no single-call form.
- **`1006620003` means the shortcut is already pinned.** It is the one error
  code worth special-casing, because it is a normal user outcome rather than a
  fault.

The nested try/catch plus a catch on both promises is the most complete error
handling in this industry's samples - copy the structure, then fix the silent
branches.

**Receiving the tap — `entryability/EntryAbility.ets`** (as shipped)

```typescript
onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {
  this.context.getApplicationContext().setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT);
  if (want.parameters?.shortCutKey === 'MainPage') {
    AppStorage.setOrCreate('shortCutKey', 'MainPage');
  }
}

onNewWant(want: Want, launchParam: AbilityConstant.LaunchParam): void {
  if (want.parameters?.shortCutKey === 'MainPage') {
    AppStorage.setOrCreate('shortCutKey', 'MainPage');
  }
}
```

Publishing through `AppStorage` rather than reaching into the page is the right
call - the ability has no reference to the UI, and the page picks it up
declaratively.

**Acting on it — `pages/MainPage.ets`** (corrected, see `HW-06-0035`)

```typescript
@StorageLink('shortCutKey') @Watch('jumpToMap') shortCutKey: string = '';

aboutToAppear(): void {
  this.jumpToMap();     // covers the cold start, where the watch may not fire
}

jumpToMap(): void {
  if (this.shortCutKey !== 'MainPage') {
    return;
  }
  this.shortCutKey = '';                              // clear first: this re-enters
  this.pageInfos.pushPathByName('MapPage', null);     // FIX: shipped body stops above
}
```

Clearing the flag **before** navigating matters: the handler watches the
variable it writes, so the assignment re-enters `jumpToMap` once. Clearing first
makes the re-entry hit the early return.

## Permissions & config

**None.** The sample declares no `requestPermissions` - pinning a shortcut for
your own app needs no permission; the launcher arbitrates.

`module.json5` also declares `"routerMap": "$profile:route_map"`, which is how
`pushPathByName('MapPage', …)` resolves the destination without a
`navDestination` builder in the page.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. The industry's architecture practice
  requires API 24.
- The `Want` passed to `checkPinShortcutPermitted` must match the one in
  `shortcuts_config.json`. A mismatch surfaces only as a runtime rejection.
- `ohos.ability.shortcuts` is a fixed metadata name.
- The launcher decides whether the pin succeeds; the app cannot force it.
- `productViewManager` comes from Store Kit, so the capability depends on the
  app distribution environment being present.

## Pitfalls

- **`HW-06-0035` — the shortcut does not open the map.** The whole chain is
  wired correctly and then `jumpToMap()` only assigns an empty string. The only
  route to `MapPage` is a double-tap gesture on the image, which the shortcut
  never reaches. The handler also writes the variable it watches.
- **`HW-06-0036` — the bundle name is hardcoded** in `MainPage.ets` and repeated
  in the profile, with nothing keeping them in step. It is the first value a
  developer changes when adapting the sample.
- **`HW-06-0037` — only `1006620003` reaches the user.** Every other failure,
  in either call, is logged and nothing else, so a failed add is
  indistinguishable from a slow success.
- **`HW-06-0038` — the variable is named `uiContext` but holds a
  `UIAbilityContext`,** which is the only type hint the snippet gives for the
  API's first parameter.

## References

- `documentation/harmonyos-references/06_application-services/store-productviewmanager.md` - `checkPinShortcutPermitted`, `requestNewPinShortcut`, `CheckShortcutResult`, error codes
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/store-productviewmanager
- `documentation/harmonyos-guides/01_getting-started/typical-scenario-configuration.md` - declaring static shortcuts in `shortcuts_config.json` and the `ohos.ability.shortcuts` metadata
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/typical-scenario-configuration
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `pushPathByName` and `routerMap`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `TRANS-03` - the widget alternative; compare the two before choosing
- `TRANS-01` - the industry architecture
