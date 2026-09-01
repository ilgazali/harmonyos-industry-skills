---
id: COMMON-33
title: Custom font in an H5 page - discover packaged .ttf files and register them into the web page with FontFace
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/33_h5_load_custom_font_library.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5_load_custom_font_library-0000002349762829
sample: huawei_industry_tree/19_common_technical_solutions/downloads/LoadLocalFontLibForH5.zip
kits: ["@kit.ArkWeb", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: ["resourceManager.getRawFileList", "webview.WebviewController", "WebviewController.runJavaScript", "Web.fileAccess", "Web.javaScriptAccess", "Web.domStorageAccess", "Web.geolocationAccess", "resource://rawfile", "component .expandSafeArea", "@ComponentV2", "@Local"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0100, HW-19-0101, HW-19-0182]
status: verified-with-fixes
---

## When to use

Load this card when an embedded H5 page has to render in a **font that ships with
the application rather than with the system** - a reading app offering a
different typeface, an accessibility mode with a high-legibility font, a brand
font on a content page.

The technique has two halves that are useful independently: enumerating the
packaged font files at runtime with `getRawFileList`, and registering a font into
a live web page with the DOM `FontFace` API driven from `runJavaScript`.

## Feature checklist

The application must:

- Ship the font files in `resources/rawfile` alongside the HTML page.
- Enumerate them at runtime rather than hardcoding names, filtering by extension.
- Derive a font family name and a page-relative URL from each file name.
- Keep a default entry representing the page's own font.
- On selection, inject a `FontFace` load into the page and add the result to
  `document.fonts`.
- Apply the family to the target elements **after** the load has resolved
  (HW-19-0100).
- Track which families are already registered so the load is not repeated - based
  on success, not on having started (HW-19-0100).
- Cycle through every discovered font, not just the first one (HW-19-0101).
- Keep the web view locked down: no file-system access, no geolocation.

## Architecture

Single-module project (`entry` HAP), one page:

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup; publishes `topRectHeight` |
| `common/Constants.ets` | all the layout numbers |
| `model/FontFamily.ets` | `{ fontName, fontUrl, isLoaded }` |
| `pages/LoadLocalFont.ets` | discovery, the `Web` component, and the switch handler |
| `resources/rawfile/news.html` | the article page plus two tiny caller functions |
| `resources/rawfile/MaoKenWangXingYuan-2.ttf` | the packaged font |

**The page is loaded as a rawfile resource**, not as a file path:

```ts
Web({ src: 'resource://rawfile/news.html', controller: this.controller })
```

This matters for the font URL: because the document's own base is the rawfile
directory, a sibling font is addressable from inside the page as `./Name.ttf` -
which is exactly what the discovery loop builds. No file-system access is needed,
and the sample says so in a comment on the very next line:
`.fileAccess(false) //若无必要启用本地文件访问，则必须显式调用fileAccess(false)禁用本地文件访问`
("if local file access is not necessary, fileAccess(false) must be called
explicitly to disable it").

**The injection protocol is two-step, by design.** `runJavaScript` evaluates a
script in the page; the sample uses it to *define* a function and then calls a
pre-existing caller in the page to *invoke* it:

`LoadLocalFontLibForH5.zip#...#/resources/rawfile/news.html`

```js
function fontChangeCaller() {
    changeFont();
}
function fontLoadCaller() {
    loadNewFont()
}
```

So the native side sends `function loadNewFont() {...}` and then
`fontLoadCaller()`. The indirection exists because a function declaration
injected by `runJavaScript` has to be defined before it can be called, and the
two calls are separate evaluations.

**Where the design breaks.** `loadNewFont` is asynchronous
(`font.load().then(...)`), but the four `runJavaScript` calls are issued
back-to-back with nothing awaiting the load - see HW-19-0100.

## Implementation steps

1. **Ship the fonts in `rawfile`** next to the HTML page, so a page-relative
   `./Name.ttf` resolves.
2. **Seed the list with the page's default font** - an entry with an empty URL
   that means "no custom family".
3. **Enumerate the directory** with
   `resourceManager.getRawFileList('', (error, value) => ...)` inside a
   `try/catch`, checking `error` before using `value`.
4. **Filter and map**: keep `.ttf`, split the name at the dot for the family, and
   prefix `./` for the URL.
5. **Load the page** with `resource://rawfile/...`, `javaScriptAccess(true)`,
   `fileAccess(false)`, `geolocationAccess(false)`.
6. **Put the caller shims in the page** so injected definitions can be invoked.
7. **On switch**, advance through the whole list (HW-19-0101) and inject the
   loader, chaining the style change onto the resolved load rather than firing it
   separately (HW-19-0100).
8. **Mark the family loaded only on success**, so a broken font file can be
   retried.

## Verified snippets

All snippets below come from the sample project, not from the document.

### Discovering the packaged fonts

`LoadLocalFontLibForH5.zip#LoadLocalFontLibForH5/entry/src/main/ets/pages/LoadLocalFont.ets`

```ts
aboutToAppear(): void {
  this.fontFamilies.push(new FontFamily('鸿蒙黑体', '')); // 页面默认字体
  // 加载本地已有的字体库
  try {
    let context = this.getUIContext().getHostContext();
    context!.resourceManager.getRawFileList('', (error: BusinessError, value: Array<string>) => {
      if (error != null) {
        hilog.info(DOMAIN, 'testTag',
          `callback getRawFileList failed, error code: ${error.code}, message: ${error.message}.`);
        return;
      }
      for (let i = 0; i < value.length; i++) {
        if (!value[i].endsWith('.ttf')) {
          continue;
        }
        let fontName = value[i].split('.')[0];
        let fontUrl = './' + value[i];
        this.fontFamilies.push(new FontFamily(fontName, fontUrl));
      }
    });
  } catch (error) {
    let code = (error as BusinessError).code;
    let message = (error as BusinessError).message;
    hilog.info(DOMAIN, 'testTag', `callback getRawFileList failed, error code: ${code}, message: ${message}.`);
  }
}
```

### The font model

`LoadLocalFontLibForH5.zip#LoadLocalFontLibForH5/entry/src/main/ets/model/FontFamily.ets`

```ts
export class FontFamily {
  fontName: string = '';
  fontUrl: string = '';
  isLoaded: boolean = false;

  constructor(name: string, url: string) {
    this.fontName = name;
    this.fontUrl = url;
  }
}
```

### The switch handler (as shipped - see HW-19-0100 and HW-19-0101)

`LoadLocalFontLibForH5.zip#LoadLocalFontLibForH5/entry/src/main/ets/pages/LoadLocalFont.ets`

```ts
Image($r('app.media.font_flag'))
  .onClick(() => {
    this.fontIndex = 1 - this.fontIndex;          // FIX (HW-19-0101): cycle the whole list
    let fontFamily = this.fontFamilies[this.fontIndex];
    // 首先加载字体
    if (!fontFamily.isLoaded) {
      let fontLoad = `function loadNewFont() {
                        const font = new FontFace('${fontFamily.fontName}', 'url(${fontFamily.fontUrl})')
                        font.load().then(function() {document.fonts.add(font)})}`;
      this.controller.runJavaScript(fontLoad);
      this.controller.runJavaScript('fontLoadCaller()');
      fontFamily.isLoaded = true;                 // FIX (HW-19-0100): set on success only
    }

    // 字体文件加载完成后执行字体切换
    let changeFont = `function changeFont(){
                          document.getElementById('news-text').style.fontFamily = '${fontFamily.fontName}'
                          document.getElementById('news-title').style.fontFamily = '${fontFamily.fontName}'
                          document.getElementById('news-time').style.fontFamily = '${fontFamily.fontName}'}`;
    this.controller.runJavaScript(changeFont);
    this.controller.runJavaScript('fontChangeCaller()');   // FIX (HW-19-0100): runs before the load resolves
  });
```

Corrected injection - one script, the switch chained onto the resolved load:

```ts
const script = `
  (function() {
    const font = new FontFace('${fontFamily.fontName}', 'url(${fontFamily.fontUrl})');
    return font.load().then(function() {
      document.fonts.add(font);
      ['news-text', 'news-title', 'news-time'].forEach(function(id) {
        document.getElementById(id).style.fontFamily = '${fontFamily.fontName}';
      });
      return 'ok';
    }).catch(function(e) { return 'fail: ' + e; });
  })()`;
this.controller.runJavaScript(script, (err, result) => {
  if (!err && result && result.indexOf('ok') >= 0) {
    fontFamily.isLoaded = true;
  }
});
```

### The web view

`LoadLocalFontLibForH5.zip#LoadLocalFontLibForH5/entry/src/main/ets/pages/LoadLocalFont.ets`

```ts
Web({ src: 'resource://rawfile/news.html', controller: this.controller })
  .fileAccess(false)  //若无必要启用本地文件访问，则必须显式调用fileAccess(false)禁用本地文件访问
  .geolocationAccess(false)
  .javaScriptAccess(true)
  .domStorageAccess(true)
  .width(Constants.ROW_WIDTH)
  .margin({ left: Constants.SPACE_16, right: Constants.SPACE_16 });
```

### The page-side caller shims

`LoadLocalFontLibForH5.zip#LoadLocalFontLibForH5/entry/src/main/resources/rawfile/news.html`

```js
function fontChangeCaller() {
    changeFont();
}
function fontLoadCaller() {
    loadNewFont()
}
```

## Permissions & config

**No permissions are required** and none are declared - the page and the fonts
are both packaged resources, so nothing is fetched from the network and no
file-system access is needed.

`LoadLocalFontLibForH5.zip#LoadLocalFontLibForH5/entry/src/main/module.json5`
declares `"deviceTypes": ["phone", "tablet", "2in1"]`, the usual `EntryAbility`
and an `EntryBackupAbility`, and no `requestPermissions` block.

Note the deliberate `.fileAccess(false)` with its explanatory comment - this is
the one sample in this industry that gets that attribute right and says why
(contrast HW-19-0089 in COMMON-28).

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`resource://rawfile/...` is what makes the relative font URL work.** Loading
  the same page through a file path would change the document base and break
  `./Name.ttf`.
- **`javaScriptAccess(true)` is mandatory** - the whole mechanism is injected
  script.
- **`runJavaScript` evaluates, it does not await.** A definition and its call are
  two separate evaluations, and neither waits for asynchronous work started
  inside the page.
- **`FontFace.load()` is a promise.** `document.fonts.add` runs only after it
  resolves; applying `style.fontFamily` before that silently falls back.
- **`getRawFileList('')` lists the whole rawfile root**, including the HTML page
  and images - hence the extension filter.
- **The family name is derived from the file name** by splitting at the first dot,
  so `MaoKenWangXingYuan-2.ttf` becomes the family `MaoKenWangXingYuan-2`. A file
  name containing a dot would be truncated.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **Applying the font family without waiting for the load is incorrect.**
  `font.load().then(...)` resolves after the four `runJavaScript` calls have
  already been issued, so the first tap can restyle to a family that is not yet in
  `document.fonts`. Chain the style change onto the resolved load - the code's own
  comment already claims this ordering. (HW-19-0100)
- **`isLoaded = true` immediately after starting the load is incorrect.** A
  failing `FontFace.load()` - the injected promise has no rejection handler - is
  recorded as a success and never retried. (HW-19-0100)
- **`this.fontIndex = 1 - this.fontIndex` is incorrect for a list built by
  scanning a directory.** Any second `.ttf` dropped into `rawfile` is discovered
  and then unreachable. Use a modulo cycle. (HW-19-0101)
- **`@Local topRectHeight = AppStorage.get('topRectHeight')` is read once and
  never used.** `@Local` does not track AppStorage, and the value does not appear
  in `build()` - dead state.
- **The injected script interpolates the family name and URL as raw text.** Both
  come from packaged file names here, so they are developer-controlled; if the
  list ever gains a remotely supplied entry, escape it before interpolation.
- **The three element ids are hardcoded** (`news-text`, `news-title`,
  `news-time`). A page whose ids change silently stops responding to the switch,
  because `runJavaScript` reports nothing when `getElementById` returns null.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-webview-webviewcontroller.md` -
  `runJavaScript` and its callback/promise forms.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/arkts-apis-webview-webviewcontroller
- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-attributes.md` -
  `fileAccess` (default false since API 12), `javaScriptAccess`,
  `domStorageAccess`, `geolocationAccess`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-basic-components-web-attributes
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` -
  `getRawFileList` and its callback contract.
  https://developer.huawei.com/consumer/cn/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-guides/03_application-framework/web-page-loading-with-web-components.md` -
  loading local pages, including the `resource://rawfile` form.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/web-page-loading-with-web-components
- `documentation/harmonyos-guides/01_getting-started/resource-categories-and-access.md` -
  the rawfile directory and how it is addressed.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/resource-categories-and-access
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/h5_load_custom_font_library-0000002349762829
