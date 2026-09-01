---
id: OFFICE-04
title: Secure online PDF preview - Web component plus canvas watermark, privacy window and a custom toolbar
industry: 05_office
doc: huawei_industry_tree/05_office/docs/04_load_display_pdf.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/load_display_pdf-0000002270470565
sample: huawei_industry_tree/05_office/downloads/LoadDisplayPDF.zip
kits: ["@kit.ArkWeb", "@kit.ArkUI", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [Web, "web_webview.WebviewController", "Web.overlay", "Web.mixedMode", "Web.javaScriptAccess", "Web.domStorageAccess", "Web.fileAccess", "Web.geolocationAccess", Canvas, CanvasRenderingContext2D, RenderingContextSettings, "CanvasRenderingContext2D.fillText", "CanvasRenderingContext2D.translate", "CanvasRenderingContext2D.rotate", HitTestMode, "windowStage.getMainWindow", "window.setWindowPrivacyMode", "window.setWindowLayoutFullScreen", "window.getWindowAvoidArea", "AppStorage.setOrCreate", "AppStorage.get", Navigation, NavPathStack, NavDestination, "NavPathStack.pushPathByName", routerMap, "UIContext.px2vp", "UIContext.getPromptAction"]
permissions: ["ohos.permission.INTERNET", "ohos.permission.PRIVACY_WINDOW"]
min_api: 20
modules: [entry]
findings: [HW-05-0018, HW-05-0019, HW-05-0020, HW-05-0021, HW-05-0185]
status: verified-with-fixes
---

## When to use

Load this card when an office app must **show a remote PDF to the user but not
let them keep it**: a product spec shown to a customer, a contract draft, an
internal report. The requirement is not "render a PDF" - it is "render a PDF
with three simultaneous leak controls":

| Control | Mechanism |
| --- | --- |
| Attribution watermark | a `Canvas` drawn as the `Web` component's `overlay` |
| Anti screenshot / screen recording | `window.setWindowPrivacyMode(true)` under `ohos.permission.PRIVACY_WINDOW` |
| Anti download | hide the built-in PDF toolbar with `#toolbar=0` and supply a custom top bar whose download button refuses |

No PDF Kit, no local file, no download at all: the document is loaded straight
into a `Web` component from an HTTPS URL.

## Feature checklist

The application must be able to:

- List the available documents and open one by name through a `Navigation` route
  declared in `routerMap`.
- Load the PDF into a `Web` component directly from its remote URL.
- Hide the PDF viewer's built-in toolbar (which carries a download button) by
  appending `#toolbar=0` to the URL.
- Draw a repeating, rotated, semi-transparent watermark over the whole viewer
  without intercepting touch events.
- Turn the window into a privacy window while the preview page is on screen, and
  turn it back off when the page is left.
- Replace the hidden toolbar with a custom top bar carrying back, title and a
  download button that shows a "prohibited" toast.
- Lock down the `Web` component: no mixed content, no file access, no geolocation.
- Lay out under the status bar using the avoid-area height published by
  `EntryAbility`.

## Architecture

Single `entry` HAP, five source files:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | full-screen layout, publishes `topHeight` and the `windowClass` object into `AppStorage`, loads `pages/MainPage` |
| `pages/MainPage.ets` | `@Entry`; owns the `NavPathStack` and the document list; `@Provide`s `title` and `path` |
| `pages/Preview.ets` | the `NavDestination`; privacy mode on enter / off on leave, custom top bar, the `Web` component |
| `components/WaterMark.ets` | `WaterMarkView` struct plus the exported `createWaterMarkView` global `@Builder` |
| `common/Constants.ets`, `common/DataType.ets` | layout constants, the watermark text, the document list, `PdfItemType` |

Two state channels are used deliberately:

- **`AppStorage`** carries what the ability owns and the pages consume:
  `windowClass` (needed for `setWindowPrivacyMode` from inside a page) and
  `topHeight` (the status-bar avoid area).
- **`@Provide` / `@Consume`** carries the selected document from `MainPage` to
  `Preview`: `title` and `path` are provided at the `Navigation` root and consumed
  in the `NavDestination`, so `pushPathByName('Preview', null)` passes no
  parameter at all.

Navigation is declared, not hard-coded: `module.json5` sets
`"routerMap": "$profile:route_map"` and `route_map.json` binds the route name
`Preview` to the global `@Builder previewBuilder` in `pages/Preview.ets`.

Flow when a document is tapped:

```
MainPage.PdfView onClick
  -> this.title = item.title; this.path = item.path      (@Provide)
  -> pageInfos.pushPathByName('Preview', null)
       -> route_map.json -> previewBuilder() -> Preview()
            aboutToAppear   -> windowClass.setWindowPrivacyMode(true)
            build           -> TabView() + Web(src: this.path).overlay(createWaterMarkView())
            aboutToDisappear-> windowClass.setWindowPrivacyMode(false)
```

## Implementation steps

1. **Declare the two permissions** in `entry/src/main/module.json5`:
   `ohos.permission.INTERNET` for the remote load and
   `ohos.permission.PRIVACY_WINDOW` for the screenshot block. Without the second
   one, `setWindowPrivacyMode` fails with error `201`.
2. **Declare the route.** Add `"routerMap": "$profile:route_map"` to
   `module.json5` and create `resources/base/profile/route_map.json` mapping the
   route name to the page's global `@Builder`.
3. **Publish the window handle and the inset.** In `onWindowStageCreate`, await
   `windowStage.getMainWindow()`, `await setWindowLayoutFullScreen(true)`
   (HW-05-0020), read `getWindowAvoidArea(TYPE_SYSTEM).topRect.height` into
   `AppStorage['topHeight']`, and store the window object under
   `AppStorage['windowClass']`. Check `err.code` in the callback before storing
   (HW-05-0021), and pass all four arguments to `hilog` (HW-05-0018).
4. **Append `#toolbar=0` to the URL.** In the sample this lives in the string
   resource itself (`"value": "https://example.com/test.pdf#toolbar=0"`), so the
   viewer's own toolbar - and its download button - never renders.
5. **Build the custom top bar.** Back arrow calls `pageInfos.pop()`; the download
   icon calls `getPromptAction().showToast({ message: $r('app.string.Prohibited') })`.
   Pad the bar with `px2vp(AppStorage.get<number>('topHeight'))`.
6. **Load the PDF with a locked-down `Web`.** `Web({ src: this.path, controller })`
   with `mixedMode(MixedMode.None)` (no HTTP inside an HTTPS page),
   `fileAccess(false)` and `geolocationAccess(false)`. `javaScriptAccess(true)` and
   `domStorageAccess(true)` stay on because the built-in PDF viewer needs them.
7. **Draw the watermark as an overlay.** Write a `WaterMarkView` struct that owns
   a `CanvasRenderingContext2D`, and export a **global** `@Builder`
   (`createWaterMarkView`) that instantiates it with
   `.hitTestBehavior(HitTestMode.None)`. Attach it with
   `.overlay(createWaterMarkView())` on the `Web`. Both the `Canvas` inside the
   struct and the struct instance in the builder must opt out of hit testing
   (`HitTestMode.Transparent` and `HitTestMode.None`), otherwise the overlay
   swallows the scroll gestures meant for the PDF.
8. **Tile the text in `onReady`.** Set `fillStyle`, `font`, `textAlign: 'center'`
   and `textBaseline: 'middle'`, then walk the canvas in `SECTOR`-sized steps:
   `translate` horizontally in the outer loop, and in the inner loop `rotate`,
   `fillText`, `rotate` back, `translate` vertically. Undo the vertical
   translation at the end of each column.
9. **Toggle privacy mode around the page.** `setWindowPrivacyMode(true)` in
   `aboutToAppear`, `false` in `aboutToDisappear`. Wrap both in `try/catch` and
   handle the promise rejection - the sample does neither (HW-05-0019).

## Verified snippets

All snippets below are copied from the sample project, not from the document.

### Privacy mode around the preview page

`LoadDisplayPDF.zip#entry/src/main/ets/pages/Preview.ets`

```ts
import web_webview from '@ohos.web.webview';
import { window } from '@kit.ArkUI';
import { createWaterMarkView } from '../components/WaterMark';

@Builder
export function previewBuilder() {
  Preview();
}

@Component
struct Preview {
  webController: web_webview.WebviewController = new web_webview.WebviewController();
  @Consume('pageInfos') pageInfos: NavPathStack;
  @Consume title: ResourceStr;
  @Consume path: ResourceStr;
  windowClass: window.Window | null | undefined = AppStorage.get('windowClass');
  isPrivacyMode: boolean = false;

  // Setting the screenshot prevention screen recording
  aboutToAppear(): void {
    this.isPrivacyMode = true;
    if (this.windowClass) {
      this.windowClass.setWindowPrivacyMode(this.isPrivacyMode); // see HW-05-0019
    }
  }

  // Exit Restore
  aboutToDisappear(): void {
    this.isPrivacyMode = false;
    if (this.windowClass) {
      this.windowClass.setWindowPrivacyMode(this.isPrivacyMode);
    }
  }
}
```

Corrected form, following the reference example:

```ts
aboutToAppear(): void {
  if (!this.windowClass) {
    return;
  }
  try {
    this.windowClass.setWindowPrivacyMode(true)
      .then(() => hilog.info(0x0000, 'testTag', 'privacy mode on'))
      .catch((err: BusinessError) =>
        hilog.error(0x0000, 'testTag', 'setWindowPrivacyMode failed: %{public}s', JSON.stringify(err)));
  } catch (exception) {
    hilog.error(0x0000, 'testTag', 'setWindowPrivacyMode threw: %{public}s', JSON.stringify(exception));
  }
}
```

### Locked-down Web component with the watermark overlay

`LoadDisplayPDF.zip#entry/src/main/ets/pages/Preview.ets`

```ts
// Loading Online PDFs
@Builder
LoadPdf() {
  Web({ src: this.path, controller: this.webController })
    .mixedMode(MixedMode.None)
    .javaScriptAccess(true)
    .domStorageAccess(true)
    .fileAccess(false)
    .geolocationAccess(false)
    .overlay(createWaterMarkView())
    .width(Constants.FULL_WIDTH)
    .height(Constants.MAIN_CONTEXT_HEIGHT);
}
```

### Custom top bar with the refusing download button

`LoadDisplayPDF.zip#entry/src/main/ets/pages/Preview.ets`

```ts
@Builder
TabView() {
  Row() {
    Row() {
      Image($r('app.media.back'))
        .width(Constants.TOP_IMAGE_WIDTH)
        .aspectRatio(1)
        .onClick(() => {
          this.pageInfos.pop();
        });
      Text(this.title)
        .fontSize(Constants.BACK_TITLE_SIZE)
        .fontWeight(Constants.FONT_WEIGHT_TITLE);
    };

    Image($r('app.media.download'))
      .width(Constants.TOP_IMAGE_WIDTH)
      .aspectRatio(1)
      .onClick(() => {
        this.getUIContext().getPromptAction().showToast({ message: $r('app.string.Prohibited') });
      });
  }
  .width(Constants.FULL_WIDTH)
  .height(Constants.HEADER_HEIGHT)
  .padding({ top: this.getUIContext().px2vp(AppStorage.get<number>('topHeight')) })
  .justifyContent(FlexAlign.SpaceBetween)
  .backgroundColor(Constants.BACKGROUND_COLOR);
}
```

### The canvas watermark

`LoadDisplayPDF.zip#entry/src/main/ets/components/WaterMark.ets`

```ts
import { Constants, MARK } from '../common/Constants';

@Component
struct WaterMarkView {
  private waterMarkSettings: RenderingContextSettings = new RenderingContextSettings(true);
  private waterMarkContext: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.waterMarkSettings);

  build() {
    Canvas(this.waterMarkContext)
      .width(Constants.FULL_WIDTH)
      .height(Constants.FULL_HEIGHT)
      .hitTestBehavior(HitTestMode.Transparent)
      .onReady(() => {
        this.waterMarkContext.fillStyle = Constants.FILL_STYLE; // '#10000000'
        this.waterMarkContext.font = Constants.FONT;            // '20vp'
        this.waterMarkContext.textAlign = 'center';
        this.waterMarkContext.textBaseline = 'middle';
        for (let i = 0; i < this.waterMarkContext.width / Constants.SECTOR; i++) {
          this.waterMarkContext.translate(Constants.SECTOR, Constants.ZERO);
          let j = 0;
          for (; j < this.waterMarkContext.height / Constants.SECTOR; j++) {
            this.waterMarkContext.rotate(-Math.PI / Constants.ROTATE_ONE * Constants.ROTATE_TWO);
            this.waterMarkContext.fillText(MARK, -60, -60);
            this.waterMarkContext.rotate(Math.PI / Constants.ROTATE_ONE * Constants.ROTATE_TWO);
            this.waterMarkContext.translate(Constants.ZERO, Constants.SECTOR);
          }
          this.waterMarkContext.translate(Constants.ZERO, -Constants.SECTOR * j);
        }
      });
  }
}

@Builder
export function createWaterMarkView() {
  WaterMarkView()
    .hitTestBehavior(HitTestMode.None); // The touch test must be set to None or Transparent
}
```

Relevant constants (`LoadDisplayPDF.zip#entry/src/main/ets/common/Constants.ets`):

```ts
export const MARK = '水印水印';

static readonly FILL_STYLE = '#10000000'; // alpha 0x10 black
static readonly FONT = '20vp';
static readonly SECTOR = 120;             // tile pitch
static readonly ROTATE_ONE = 180;
static readonly ROTATE_TWO = 30;          // -30 degrees
```

### Window handle and inset published at startup

`LoadDisplayPDF.zip#entry/src/main/ets/entryability/EntryAbility.ets`

```ts
async onWindowStageCreate(windowStage: window.WindowStage): Promise<void> {
  this.context.getApplicationContext().setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT);
  const WIN = await windowStage.getMainWindow();
  WIN.setWindowLayoutFullScreen(true); // should be awaited - HW-05-0020
  AppStorage.setOrCreate(
    'topHeight',
    WIN.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM).topRect.height
  );

  // Obtains and saves the windowClass object
  windowStage.getMainWindow((err: BusinessError, data) => {
    let errCode: number = err.code;
    if (errCode) {
      hilog.error(0x0000, 'Failed to obtain the main window. Cause: %{public}s', JSON.stringify(err)); // HW-05-0018
      return;
    }
    const windowClass = data;
    AppStorage.setOrCreate('windowClass', windowClass);
  });

  windowStage.loadContent('pages/MainPage', (err) => {
    if (err.code) {
      hilog.error(0x0000, 'testTag', 'Failed to load the content. Cause: %{public}s', JSON.stringify(err) ?? '');
      return;
    }
  });
}
```

### Route declaration

`LoadDisplayPDF.zip#entry/src/main/resources/base/profile/route_map.json`

```json
{
  "routerMap": [
    {
      "name": "Preview",
      "pageSourceFile": "src/main/ets/pages/Preview.ets",
      "buildFunction": "previewBuilder",
      "data": {
        "description": "this is PageOne"
      }
    }
  ]
}
```

## Permissions & config

`LoadDisplayPDF.zip#entry/src/main/module.json5`

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "pages": "$profile:main_pages",
    "routerMap": "$profile:route_map",
    "requestPermissions": [
      {
        "name": "ohos.permission.INTERNET",
        "reason": "$string:EntryAbility_desc",
        "usedScene": { "abilities": ["EntryAbility"], "when": "always" }
      },
      {
        "name": "ohos.permission.PRIVACY_WINDOW"
      }
    ]
  }
}
```

Notes on the config:

- The document's 权限说明 section lists exactly `ohos.permission.INTERNET` and
  `ohos.permission.PRIVACY_WINDOW`, and the sample declares exactly those two -
  verified consistent.
- `PRIVACY_WINDOW` is declared with `name` only. Per the official
  declare-permissions guide, `reason` and `usedScene` are mandatory only for
  `user_grant` and `manual_settings` permissions and optional otherwise, so the
  short form is acceptable here.
- `routerMap` must be present for `pushPathByName('Preview', ...)` to resolve;
  the route's `buildFunction` must name a **global** `@Builder` exported from the
  page file, which is why `previewBuilder` sits outside the struct.
- `build-profile.json5` pins `compatibleSdkVersion` and `targetSdkVersion` to
  `6.0.0(20)` and enables
  `strictMode: { caseSensitiveCheck: true, useNormalizedOHMUrl: true }`.

## Constraints

- **Versions.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later; `compatibleSdkVersion` is
  `6.0.0(20)`.
- **Devices.** `deviceTypes` is `["phone", "tablet", "2in1"]`;
  `setWindowPrivacyMode` is documented for Phone, PC/2in1, Tablet, TV and
  Wearable.
- **The URLs are placeholders.** Both entries in `Constants.ets` resolve to
  `https://example.com/test.pdf#toolbar=0`, and the document says so explicitly:
  "效果预览仅为演示，真实运行需要替换为自己的URL" ("The preview is a demonstration
  only; a real run requires replacing it with your own URL."). Nothing in this
  sample proves any particular PDF renders.
- **`#toolbar=0` is a viewer hint, not a security boundary.** It hides the
  built-in toolbar of the PDF viewer; it does not prevent a determined user from
  fetching the URL directly. The real controls are the privacy window and
  whatever authorization sits in front of the URL.
- **Privacy mode is window-scoped, not page-scoped.** The sample stores the main
  window in `AppStorage` and toggles it in the page lifecycle, so any other page
  shown in the same window while `Preview` is alive is also protected - and any
  early return from `aboutToDisappear` would leave the whole app in privacy mode.
- **Privacy mode can fail at runtime.** Error `201` is returned when
  `ohos.permission.PRIVACY_WINDOW` is not effectively held, and `1300002` when
  the window state is abnormal. Both need handling before the document is shown.
- **The watermark overlay must not take touch.** `HitTestMode.Transparent` on the
  `Canvas` and `HitTestMode.None` on the struct instance are both required; the
  sample comments this explicitly.

## Pitfalls

- **`hilog.error(0x0000, 'Failed to obtain the main window. Cause: %{public}s', JSON.stringify(err))`
  is incorrect.** The signature is `error(domain, tag, format, ...args)`, so the
  message lands in the 31-byte `tag` slot and the serialized error becomes the
  format string; the placeholder never expands. Pass four arguments, as the same
  file does three lines later. (HW-05-0018)
- **Calling `setWindowPrivacyMode` bare is incorrect.** It returns a
  `Promise<void>` and can also throw; a `201` permission failure therefore leaves
  the PDF screenshot-able with no signal at all. Wrap it in `try/catch` and handle
  the rejection, as the official example does. (HW-05-0019)
- **`WIN.setWindowLayoutFullScreen(true)` without `await` is incorrect** in an
  `async` method that already awaits the line above: the rejection is unhandled
  and `getWindowAvoidArea` on the next line may read the pre-full-screen inset.
  (HW-05-0020)
- **The document's `getMainWindow` snippet ignores `err`, which is incorrect.**
  On failure it stores `undefined` under `AppStorage['windowClass']`, and
  `Preview`'s `if (this.windowClass)` guard then silently skips the privacy-mode
  call. Check `err.code` and return early, as the sample does. (HW-05-0021)

## References

Reference pages used to verify this card:

- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` -
  `setWindowPrivacyMode9+` (required permission `ohos.permission.PRIVACY_WINDOW`,
  error codes 201 and 1300002, try/catch + promise example) and
  `setWindowLayoutFullScreen`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-references/03_system/js-apis-hilog.md` -
  `error(domain, tag, format, ...args)` and the 31-byte tag limit.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-hilog
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - when
  `reason` and `usedScene` are mandatory in `requestPermissions`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
