---
id: SPORT-09
title: Publish a group activity - RichEditor to HTML, cascading sheets, photo picker
industry: 03_sports_health
doc: huawei_industry_tree/03_sports_health/docs/09_outdoor_sports.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/outdoor_sports-0000002337059970
sample: huawei_industry_tree/03_sports_health/downloads/OutdoorSports.zip
kits: ["@kit.ArkUI", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: [RichEditor, RichEditorController, RichEditorOptions, setTypingStyle, getSpans, RichEditorTextSpanResult, RichEditorImageSpanResult, bindSheet, Web, WebviewController, loadData, javaScriptAccess, fileAccess, mixedMode, MixedMode, metaViewport, WebLayoutMode, "photoAccessHelper.PhotoViewPicker", PhotoSelectOptions, List, ForEach, fadingEdge, LengthMetrics, "@StorageProp", "@StorageLink", NavPathStack]
permissions: ["ohos.permission.INTERNET"]
min_api: 20
modules: [entry]
findings: [HW-03-0036, HW-03-0037, HW-03-0038, HW-03-0055]
status: verified-with-fixes
---

## When to use

Load this card for **user-generated rich content** - a post with mixed text
and images that other users read. Group-activity recruitment here; the same
shape covers a review with photos, a training log, a recipe, a classified ad.

The interesting mechanism is the round trip: **`RichEditor` for authoring →
HTML string → `Web` for display**. There is no rich-text renderer in ArkUI, so
the sample converts the editor's spans to HTML and hands them to a web view.
That is a legitimate approach and the reason to read this card - and it is
also where its worst defect lives (`HW-03-0036`), because generated HTML plus
a permissive `Web` is an execution surface.

It is also the corpus's clearest example of **`bindSheet` used for cascading
pickers** - date, time and a two- or three-level region selector, each in a
half-modal sheet.

## Feature checklist

- A feed of group activities, and a tab for the ones the user created.
- A compose screen: title, date, time, region, cover image, rich description.
- Date, time and region are chosen from half-modal sheets with linked levels.
- Images come from the system gallery.
- The description is authored in a `RichEditor` mixing text and images.
- Publishing converts the content to HTML and adds it to the feed.
- Opening an activity renders that HTML in a `Web` component.

## Architecture

One `entry` module, four pages, six utilities.

```
entry/src/main/ets
├── entryability/EntryAbility.ets
├── entrybackupability/
├── pages
│   ├── Index.ets           the feed, the tabs, and the publish state
│   ├── LunchPartner.ets    the compose screen: sheets, picker, RichEditor
│   ├── Detail.ets          the reader: WebBuilder + loadData
│   └── SpanToHtml.ets      RichEditor spans -> HTML string
└── utils
    ├── BindSheet.ets       the cascading sheet builders
    ├── CommonUtils.ets     handleAddr and friends
    ├── Constants.ets       ActivityData and the demo values
    ├── EnumUtils.ets       SheetTypeEnum
    ├── FileUtil.ets        the photo picker and the sandbox copy
    └── Logger.ets
```

The documented tree matches the zip.

**The content pipeline is the architecture:**

```
RichEditor ──getSpans()──> (RichEditorTextSpanResult | RichEditorImageSpanResult)[]
                                        │
                          SpanToHtml: styles -> inline CSS
                                      images -> <img src='file://…'>
                                        │
                                   one HTML string
                                        │
                          stored in ActivityData, published to AppStorage
                                        │
                    Detail: Web().onControllerAttached -> loadData(html, 'text/html', 'UTF-8')
```

Storing the *rendered HTML* rather than the span array is the trade-off worth
understanding: it makes display trivial and makes the content un-editable
afterwards, and it moves the escaping responsibility to the producer - which
is where `HW-03-0036` bites.

## Implementation steps

1. **Author in `RichEditor`** with a `RichEditorController`, and set a typing
   style in `onReady` so new text has a defined appearance.
2. **Read the content back with `getSpans()`** and branch on whether the span
   carries a `value` (text) or not (image).
3. **Escape every piece of user text before interpolating it**
   (`HW-03-0036`).
4. **Map ArkTS colours to CSS**: ARGB to RGBA, since the two platforms order
   the alpha channel differently.
5. **Render with the narrowest `Web` configuration that works** - JavaScript
   off for static content.
6. **Bind each picker to its own sheet** with one builder switching on a sheet
   type enum.
7. **Bind the shared list two-way in the writer, one-way in the readers**
   (`HW-03-0038`).

## Verified snippets

All snippets are from `OutdoorSports.zip`. Corrected forms are marked.

**Setting up the editor — `entry/src/main/ets/pages/LunchPartner.ets`** (as shipped)

```typescript
private richEditorController: RichEditorController = new RichEditorController();
private richEditorOptions: RichEditorOptions = { controller: this.richEditorController };

RichEditor(this.richEditorOptions)
  .onReady(() => {
    this.richEditorController.setTypingStyle({
      fontWeight: FontWeight.Normal,
      fontColor: Color.Black,
      fontSize: 16,
      fontStyle: FontStyle.Normal,
      fontFamily: 'HarmonyOS Sans',
      letterSpacing: 1,
      lineHeight: 25
    })
  })
```

**`setTypingStyle` in `onReady` is what stops the first typed character
inheriting an undefined style.** Without it the spans come back with
`textStyle` fields the converter then has to default, which is exactly the
branch `SpanToHtml` needs for the `item.textStyle === undefined` case.

**Converting spans to HTML — `entry/src/main/ets/pages/SpanToHtml.ets`** (corrected, see `HW-03-0036`)

```typescript
function escapeHtml(s: string): string {          // FIX: absent in the sample
  return s.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');
}

spans.forEach(span => {
  let spanToHtml = '';
  const item = span as RichEditorTextSpanResult;
  if (item.value !== undefined) {
    const text: string = escapeHtml(item.value);   // FIX: the sample interpolates item.value raw
    if (item.value !== '\n' && item.textStyle !== undefined) {
      const size =
        `<span style="white-space: pre-wrap;` +
        (item.textStyle.fontSize !== undefined ? `font-size:${item.textStyle.fontSize}px;` : `font-size:20px;`) +
        (item.textStyle.fontColor !== undefined ? `color:${argb2Rgba(item.textStyle.fontColor.toString())};` : ``) +
        (item.textStyle.letterSpacing !== undefined ? `letter-spacing:${item.textStyle.letterSpacing};` : ``) +
        (item.textStyle.lineHeight !== undefined ? `line-height:${item.textStyle.lineHeight}px;` : `line-height: 25px;`) +
        `" >`;
      spanToHtml = size + text + '</span>';
    } else {
      spanToHtml = `<span style="white-space: pre-wrap; font-size:20px">${text}</span>`;
    }
  } else {
    const item = span as RichEditorImageSpanResult;
    const fileSandbox = getSandbox(item.valueResourceStr as string, uiContext);
    spanToHtml = `<img src='file://${fileSandbox}' width="100%", height="auto">`;
  }
  html = html + spanToHtml;
});

// ArkTS uses ARGB; CSS wants RGB or RGBA. Each platform needs its own conversion.
function argb2Rgba(argbColor: string): string {
  if (argbColor === undefined || argbColor.length !== 9) {
    return argbColor;
  }
  return '#' + argbColor.substring(3, argbColor.length) + argbColor.substring(1, 3);
}
```

**`argb2Rgba` is the detail nobody expects.** `Color` values in ArkTS
stringify as `#AARRGGBB`; CSS reads `#RRGGBBAA`. Moving the first two hex
digits to the end is the whole conversion, and skipping it turns a black
opaque text colour into a transparent red one. The length check guards the
short forms that carry no alpha.

**`white-space: pre-wrap` on every span** is what preserves the line breaks
the user typed - the editor emits `\n` inside text spans rather than separate
paragraphs, and without this the browser collapses them.

**Rendering the result — `entry/src/main/ets/pages/Detail.ets`** (corrected, see `HW-03-0036`)

```typescript
@Builder
export function WebBuilder(webviewController: WebviewController, spanToHtmlStr: string) {
  Web({ src: '', controller: webviewController })
    .onControllerAttached(() => {
      try {
        webviewController.loadData(spanToHtmlStr, 'text/html', 'UTF-8', ' ', ' ');
      } catch (error) {
        Logger.error(`ErrorCode: ${(error as BusinessError).code}, Message: ${(error as BusinessError).message}`);
      }
    })
    .metaViewport(true)
    .domStorageAccess(true)
    .fileAccess(true)                      // needed only for the file:// images
    .mixedMode(MixedMode.None)             // FIX: the sample uses MixedMode.All
    .javaScriptAccess(false)               // FIX: the sample uses true - static content needs none
    .zoomAccess(false)
    .layoutMode(WebLayoutMode.FIT_CONTENT) // the web view sizes itself to the content
    .constraintSize({ maxHeight: 'auto' })
}
```

**`src: ''` with `loadData` in `onControllerAttached`** is the pattern for
rendering a string rather than a URL - the component needs a controller before
it can be loaded, and `onControllerAttached` is the first point at which one
exists.

**`WebLayoutMode.FIT_CONTENT`** is what lets the article sit inside a
scrolling page instead of being a fixed-height scroller of its own.

**Cascading pickers — `entry/src/main/ets/pages/LunchPartner.ets`** (as shipped)

```typescript
@State isShowSheetCity: boolean = false;
@State cityAddr: string = Constants.ADDR_DEMO;

@Builder
SheetBuilder(_type: string) {
  if (_type === SheetTypeEnum.CITY_ADDR) {
    CitySheet({ isShowSheetCity: this.isShowSheetCity, cityAddr: this.cityAddr });
  }
  // ... date and time sheets on the same switch
}
```

One builder switching on an enum, with a boolean per sheet, keeps three
different pickers on one screen without three sets of near-identical
`bindSheet` code.

**Picking photos — `entry/src/main/ets/utils/FileUtil.ets`** (as shipped)

```typescript
import { photoAccessHelper } from '@kit.MediaLibraryKit';

const photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
const photoPicker = new photoAccessHelper.PhotoViewPicker();
try {
  const photoSelectResult = await photoPicker.select(photoSelectOptions);
  // ...
} catch (error) {
  hilog.error(0x0000, '[FileUtil]', `PhotoViewPicker failed with err: ${error.code}, ${error.message}`);
}
```

**`PhotoViewPicker` needs no permission** - it runs in the system gallery
process. That is why the module declares only `INTERNET`.

**The feed list — `entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
List({ space: 10, initialIndex: 0 }) {
  ForEach(this.allDataArray, (item: ActivityData) => {
    // ...
  }, (item: ActivityData, index: number) => JSON.stringify(item.title) + index)
}
.fadingEdge(true, { fadingEdgeLength: LengthMetrics.vp(40) })
```

`fadingEdge` with an explicit `LengthMetrics.vp(40)` fades the list into its
container instead of cutting the last row off - a small touch worth copying
for any feed that scrolls under a fixed header.

## Permissions & config

The sample declares one permission, with two errors in it (`HW-03-0037`).
Corrected:

```json5
// entry/src/main/module.json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" }     // system_grant: reason and usedScene are optional
]
```

No gallery permission is needed - `PhotoViewPicker` is a system picker. No
`client_id` and no map configuration: there is no map here despite the
outdoor-activity subject.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The published content is HTML, not structured data.** Once converted there
  is no path back to the span array, so an activity cannot be edited after
  publishing, and any change to the conversion does not reach existing posts.
- **Images are referenced as `file://` paths into the app sandbox**, so the
  HTML is only meaningful on the device that produced it. Sending a post to a
  server means uploading the images and rewriting the `src` attributes.
- Everything lives in `AppStorage`; nothing is persisted across a process
  restart and nothing is sent anywhere.
- The region picker's data is a static cascade in `Constants.ets`.

## Pitfalls

- **`HW-03-0036` — user text is interpolated into HTML unescaped** and
  rendered with `javaScriptAccess(true)`, `fileAccess(true)` and
  `mixedMode(MixedMode.All)`. The scenario is content one user writes and
  others read, so the input is untrusted by design.
- **`HW-03-0037` — the permission's `usedScene` names `FormAbility`,** which
  the module does not declare, and its `reason` points at the button label
  `continue_add` ("Continue Add").
- **`HW-03-0038` — the publishing component binds `allDataArray` with
  `@StorageProp`** and has to `push` then `setOrCreate` by hand, while the two
  read-only lists take `@StorageLink`. The bindings are the wrong way round.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-common-components-richeditor.md` - `RichEditor`, `RichEditorController`, `setTypingStyle`, `getSpans`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-common-components-richeditor
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-richeditor.md` - `RichEditorTextSpanResult` and `RichEditorImageSpanResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-richeditor
- `documentation/harmonyos-guides/03_application-framework/arkts-sheet-page.md` - `bindSheet` and half-modal pages
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-sheet-page
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-web.md` - `loadData`, `javaScriptAccess`, `fileAccess`, `mixedMode`, `WebLayoutMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-web
- `documentation/harmonyos-references/04_media/js-apis-photoAccessHelper.md` - `PhotoViewPicker` and `PhotoSelectOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-photoaccesshelper
- `documentation/harmonyos-guides/03_application-framework/arkts-appstorage.md` - `@StorageProp` versus `@StorageLink`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-appstorage
- `documentation/harmonyos-guides/04_system/declare-permissions.md` - `reason` and `usedScene`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/declare-permissions
- `TOUR-08` - the same `PhotoViewPicker` flow for a review form
