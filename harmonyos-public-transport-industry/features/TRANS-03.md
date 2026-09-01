---
id: TRANS-03
title: Home-screen widget that opens the ride code
industry: 06_public_transport
doc: huawei_industry_tree/06_public_transport/docs/03_qrcard_demo.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/qrcard_demo-0000002328156469
sample: huawei_industry_tree/06_public_transport/downloads/QRcardDemo.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.FormKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: [FormLink, FormExtensionAbility, form_config.json, Want.parameters, UIAbility.onCreate, UIAbility.onNewWant, WindowStage.loadContent, Window.on avoidAreaChange, ConfigurationConstant.ColorMode]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-06-0018, HW-06-0019, HW-06-0020, HW-06-0021, HW-06-0022, HW-06-0023, HW-06-0024, HW-06-0025, HW-06-0051]
status: verified-with-fixes
---

## When to use

Load this card when the app needs a **home-screen widget that launches a
specific page** when tapped - a ride code, a boarding pass, a check-in screen, a
"start my commute" shortcut. The mechanism is `FormLink` with the `router`
action, and it is the cheapest possible widget: no data binding, no refresh
cycle, just a tile that opens a page.

For a widget that must display live data, this is the wrong starting point -
this one is configured `isDynamic: false` and `updateEnabled: false`.

## Feature checklist

- Long-pressing the app icon offers a widget the user can drop on the home
  screen.
- The widget renders a branded tile: title, subtitle, icon, background image.
- Tapping anywhere on the tile launches the app.
- The launch carries a parameter naming the page to open.
- The app opens that page directly, not its default screen.

## Architecture

Single `entry` module, four moving parts:

```
entry/src/main/ets
├── pages/QrCodeCard.ets            the widget's UI (a @Entry @Component)
├── pages/QrCodePage.ets            the app page the widget targets
├── pages/QrCodeViewModel.ets
├── qrcodeformability/QrCodeFormAbility.ets   FormExtensionAbility lifecycle
└── entryability/EntryAbility.ets   receives the launch and routes
entry/src/main/resources/base/profile/form_config.json   widget declaration
```

The chain: `form_config.json` declares the widget and points `src` at
`QrCodeCard.ets` → the widget's `FormLink` fires a `router` action naming
`EntryAbility` → the system launches that ability with the params in
`want.parameters.params` → the ability reads them and decides what to show.

**The last step is where the sample stops short** - it logs the target and loads
a hardcoded page (`HW-06-0018`). The card below shows the completed form.

## Implementation steps

1. **Declare the widget** in `form_config.json` and register a
   `FormExtensionAbility` in `module.json5` pointing at it.
2. **Build the widget UI as a normal `@Entry @Component`** whose root is a
   `FormLink`.
3. **Set `action: 'router'`, `abilityName`, and a `params` object** carrying
   whatever the app needs to route.
4. **In the receiving ability, parse `want.parameters.params` inside a
   `try/catch`** (`HW-06-0020`).
5. **Use the routing value.** On cold start it selects what `loadContent`
   loads; on `onNewWant` it pushes onto the running app's stack
   (`HW-06-0018`).
6. **Validate the value against a known page list** before using it - it
   crosses a process boundary.
7. **Release `avoidAreaChange` in `onWindowStageDestroy`** (`HW-06-0021`).

## Verified snippets

All snippets are from `QRcardDemo.zip`. Corrected forms are marked.

**Widget declaration — `resources/base/profile/form_config.json`** (as shipped)

```json
{
  "forms": [
    {
      "name": "qrCode",
      "displayName": "$string:qrCode_display_name",
      "description": "$string:qrCode_desc",
      "src": "./ets/pages/QrCodeCard.ets",
      "uiSyntax": "arkts",
      "window": { "designWidth": 720, "autoDesignWidth": true },
      "colorMode": "auto",
      "isDynamic": false,
      "isDefault": true,
      "updateEnabled": false,
      "defaultDimension": "1*2",
      "supportDimensions": ["1*2"]
    }
  ]
}
```

`isDynamic: false` is the choice that defines this practice: a static card is
rendered once and cannot refresh, which is all a launcher tile needs.
`colorMode: "auto"` lets the tile follow the system theme.
`supportDimensions` limits the sizes the user can pick.

> The shipped file also sets `scheduledUpdateTime` and `updateDuration` while
> `updateEnabled` is false, so both are inert - see `HW-06-0023`.

**The widget UI — `pages/QrCodeCard.ets`** (corrected, see `HW-06-0019`)

```typescript
@Entry
@Component
export struct QrCodeCard {
  readonly actionType = 'router';
  readonly abilityName = 'EntryAbility';
  readonly message = 'add detail';

  build() {
    FormLink({
      action: this.actionType,
      abilityName: this.abilityName,
      params: {
        message: this.message,        // FIX: shipped code appends the literal
                                      // string 'app.string.widget_mine' here
        linkType: 'form_card',
        router: 'QrCodePage'
      }
    }) {
      Row({ space: Constants.TAB_SPACE }) {
        Column({ space: Constants.LIST_SPACE }) {
          Text($r('app.string.subway_go'));
          Text($r('app.string.widget_mine'));   // the correct way to use that resource
        }
        .alignItems(HorizontalAlign.Start);

        Image($r('app.media.subway'))
          .size({ width: Constants.IMAGE_WIDTH_HEIGHT, height: Constants.IMAGE_WIDTH_HEIGHT });
      }
      .justifyContent(FlexAlign.SpaceBetween)
      .backgroundImage($r('app.media.back'))
      .backgroundImageSize(ImageSize.Cover)
      .height(Constants.FULL)
      .width(Constants.FULL);
    };
  }
}
```

The shape worth copying: **`FormLink` wraps the whole tile**, so the entire card
is the tap target rather than a button inside it. Everything visual goes in its
child block.

`action` values: `'router'` opens a UIAbility, `'message'` calls back into the
`FormExtensionAbility` without launching the app, `'call'` starts the ability in
the background.

**Receiving the launch — `entryability/EntryAbility.ets`** (corrected, see `HW-06-0018`, `HW-06-0020`, `HW-06-0024`)

```typescript
private targetPage: string = 'pages/QrCodePage';

private readPage(want: Want): string | undefined {
  if (!want?.parameters?.params) {
    return undefined;
  }
  try {                                   // FIX: shipped code parses unguarded
    const params: Record<string, Object> = JSON.parse(String(want.parameters.params));
    const page = String(params.router);
    // FIX: validate - this value crosses a process boundary
    return QrCodeRoutes.has(page) ? `pages/${page}` : undefined;
  } catch (e) {
    hilog.error(DOMAIN, 'testTag', 'bad card params: %{public}s', JSON.stringify(e));
    return undefined;
  }
}

onCreate(want: Want): void {
  this.context.getApplicationContext().setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT);
  this.targetPage = this.readPage(want) ?? this.targetPage;
}

onNewWant(want: Want): void {
  const page = this.readPage(want);
  if (page) {
    // app already running: push onto the live navigation stack
    AppStorage.setOrCreate('pendingRoute', page);
  }
}

onWindowStageCreate(windowStage: window.WindowStage): void {
  // ... full-screen layout and avoid areas ...
  windowStage.loadContent(this.targetPage, (err) => {   // FIX: shipped code hardcodes the path
    if (err.code) {
      hilog.error(DOMAIN, 'testTag', 'Failed to load the content. Cause: %{public}s', JSON.stringify(err));
      return;
    }
  });
}

onWindowStageDestroy(): void {
  this.windowClass?.off('avoidAreaChange');   // FIX: shipped hook only logs
}
```

Both `onCreate` and `onNewWant` matter: the first covers a cold start from the
card, the second covers tapping the card while the app is already running. The
sample parses in both and acts on neither.

**Full-screen layout and system insets — same file** (as shipped)

```typescript
let windowClass: window.Window = windowStage.getMainWindowSync();
windowClass.setWindowLayoutFullScreen(true)
  .then(() => {
    let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR;
    let avoidArea = windowClass.getWindowAvoidArea(type);
    AppStorage.setOrCreate('bottomRectHeight', avoidArea.bottomRect.height);

    type = window.AvoidAreaType.TYPE_SYSTEM;
    avoidArea = windowClass.getWindowAvoidArea(type);
    AppStorage.setOrCreate('topRectHeight', avoidArea.topRect.height);
  })
  .catch((err: BusinessError) => {
    hilog.error(DOMAIN, 'testTag', `Failed to set full screen. Cause: ${JSON.stringify(err)}`);
  });

windowClass.on('avoidAreaChange', (data) => {
  if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
    AppStorage.setOrCreate('topRectHeight', data.area.topRect.height);
  } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
    AppStorage.setOrCreate('bottomRectHeight', data.area.bottomRect.height);
  }
});
```

Reading the avoid areas **inside** the `setWindowLayoutFullScreen` resolution is
correct - before that promise settles the window is not yet full-screen and the
areas would be measured against the wrong layout.

**Logging with hilog** (corrected, see `HW-06-0024`)

```typescript
// FIX: shipped code writes 'Target Page ' + params.router as the format string
hilog.info(DOMAIN, 'testTag', 'Target Page %{public}s', String(params.router));
```

The third argument is a **format string**; values belong in the arguments after
it, tagged `%{public}s` or `%{private}s`. Concatenating data into the format
loses that tagging and lets stray percent sequences be read as directives.

## Permissions & config

**None.** `entry/src/main/module.json5` declares no `requestPermissions` - a
widget that launches its own app needs none.

`module.json5` registers the widget's extension ability:

```json5
"extensionAbilities": [
  {
    "name": "QrCodeFormAbility",
    "srcEntry": "./ets/qrcodeformability/QrCodeFormAbility.ets",
    "label": "$string:QrCodeFormAbility_label",
    "type": "form",
    "metadata": [
      { "name": "ohos.extension.form", "resource": "$profile:form_config" }
    ]
  }
]
```

The `metadata` entry is what binds the extension ability to `form_config.json`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. **The industry's architecture practice
  requires API 24** (`HW-06-0025`), so the combined floor is 24.
- Static widget: no refresh, no live data. A ride code that must not go stale
  needs a dynamic widget and a different practice.
- Only the `1*2` dimension is offered.
- `want.parameters.params` arrives as a **string**, not an object - it must be
  parsed.
- `FormLink` cannot be nested inside another `FormLink`, and only one root-level
  action applies per tile.

## Pitfalls

- **`HW-06-0018` — the `router` parameter is never used.** Both `onCreate` and
  `onNewWant` parse it and hand it to `hilog`; `loadContent` names a fixed page.
  The demo only appears to work because the app has one page.
- **`HW-06-0019` — `'app.string.widget_mine'` is concatenated as a literal.**
  The card sends `'add detailapp.string.widget_mine'`. The same resource is used
  correctly with `$r()` two lines below.
- **`HW-06-0020` — `JSON.parse` of external want params has no `try/catch`,**
  and it runs in `onCreate`, so a malformed payload fails the launch.
- **`HW-06-0021` — `avoidAreaChange` is never released.** `onWindowStageDestroy`
  carries a comment about releasing UI resources and releases nothing.
- **`HW-06-0022` — the document calls the widget static, then dynamic.**
  `form_config.json` settles it: `isDynamic: false`. The wrong sentence is the
  one introducing the code.
- **`HW-06-0023` — `scheduledUpdateTime` and `updateDuration` are set while
  `updateEnabled` is false,** so both are inert.
- **`HW-06-0024` — parameter data is concatenated into `hilog`'s format
  string** instead of being passed as a tagged argument.
- **`HW-06-0025` — the industry declares two toolchain baselines:** API 24 for
  the architecture, API 20 for all six features, under differently named
  sections.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-formlink.md` - `FormLink`, its `action` values and `params`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-formlink
- `documentation/harmonyos-guides/03_application-framework/arkts-ui-widget-event-router.md` - the `router` event, and how params reach the ability
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-ui-widget-event-router
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `getWindowAvoidArea`, `on`/`off('avoidAreaChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `TRANS-01` - the industry architecture this widget launches into
- `TRANS-06` - the other home-screen shortcut practice in this industry; compare the two mechanisms before choosing
