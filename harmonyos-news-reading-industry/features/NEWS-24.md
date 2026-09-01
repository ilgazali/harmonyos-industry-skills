---
id: NEWS-24
title: Rich-text post editor - RichEditor spans serialised to HTML and replayed in a Web view
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/24_auto_flip_read.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/auto_flip_read-0000002384511241
sample: huawei_industry_tree/11_news_reading/downloads/SimpleArticleEdit.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.ArkWeb", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [RichEditor, RichEditorController, setTypingStyle, addImageSpan, getSpans, getCaretOffset, RichEditorTextSpanResult, RichEditorImageSpanResult, "photoAccessHelper.PhotoViewPicker", PhotoSelectOptions, "webview.WebviewController", loadData, fileAccess, layoutMode, "uri.URI", "fs.copyFileSync", Navigation, NavPathStack, KeyboardAvoidMode, "@Observed", "@Track", "@StorageLink"]
permissions: []
min_api: 20
modules: [logger (har), entry]
findings: [HW-11-0031, HW-11-0045, HW-11-0046, HW-11-0047, HW-11-0048, HW-11-0049]
status: verified
---

## When to use

Load this card when the app has to **let a user compose text and pictures
together, and then show the result somewhere else looking exactly the same**.
That "somewhere else" is what makes it hard: an editor's document model is not
a rendering format, and a detail page built out of ArkUI components will drift
from the editor's layout the moment either side changes.

The pattern here settles it by making **HTML the wire format**. `RichEditor`
composes; on publish, `getSpans()` is walked once and each span is written out
as an inline-styled `<span>` or an `<img>`; the detail page hands that string
to a `Web` component with `loadData`. One serialiser to maintain, and the
detail view is a browser rather than a second layout to keep in sync.

It generalises to any post/preview split - a review with photos, a note that
gets shared, a draft that has to survive as a string in a database - and the
two translation problems it solves are the ones you will hit every time:
ArkUI's `#AARRGGBB` colours are not CSS colours, and a picker URI is not
something a `Web` can load.

## Feature checklist

- A publish screen with a bold title field (30 characters max) over a
  full-height rich-text body.
- The body carries a typing style set once on ready: 16 fp, black, normal
  weight, `HarmonyOS Sans`, 25 vp line height, 1 vp letter spacing.
- A bottom toolbar with four entries; 上传图片 (upload image) opens the system
  gallery picker, the other three raise a "demo only" toast.
- Up to nine images can be picked at once; each is inserted as an image span at
  the current caret offset.
- Publish validates that the title and the body are both non-empty, toasts
  发布成功 (published) or 请填写相关信息 (fill in the details).
- On success the composed content is converted to an HTML document and the
  article detail page is pushed a second later.
- The detail page shows title, author row with a Follow chip, date and place,
  then the article body rendered in a `Web` at the same sizes, colours and line
  heights as the editor showed.
- Inline images render at 100% width with automatic height.
- The keyboard resizes the page rather than covering the caret.

## Architecture

Two modules: the `entry` application and a static `har` that contains nothing
but a logger.

```
common/logger                        static har: Logger wrapper over hilog, exported from Index.ets
entry/src/main/ets
├── entryability/EntryAbility.ets     full-screen window, avoid areas + keyboard height -> AppStorage,
│                                     setKeyboardAvoidMode(KeyboardAvoidMode.RESIZE)
├── entrybackupability/EntryBackupAbility.ets
├── pages/ArticleDetail.ets           NavDestination: header, the Web builder, the toolbar
├── pages/Index.ets                   @Entry: Navigation, title field, RichEditor, the four-item toolbar
├── pages/SpanToHtml.ets              SpansToHtml() + argb2Rgba() - the serialiser
└── utils
    ├── Constants.ets                 nickname, place, canned counts, toast strings
    ├── DataModel.ets                 @Observed Article with @Track articleHtml
    └── FileUtil.ets                  fileSelect() (PhotoViewPicker) + getSandbox() (copy into cacheDir)
```

The documented tree matches the zip exactly. One documentation oddity worth
knowing: the page's URL slug is `auto_flip_read` (auto page-turn) while the
page itself is 图文动态编辑发布 - rich-text publishing. The slug is wrong, not
the content; the repo path keeps the slug.

**The design decision worth copying** is that the article is a *string*, not a
component tree. `Article` carries `articleHtml: string` and the detail page's
only content contract is that string:

```typescript
WebBuilder(this.webviewController, this.article.articleHtml);
```

Everything downstream - persistence, sharing, a server round trip - inherits
that for free, and the fidelity problem collapses into one function,
`SpansToHtml`. The corollary is that the serialiser must be complete: any style
the editor can produce and the serialiser cannot express is silently lost, so
the typing style is deliberately fixed in `onReady` rather than left to the
user.

The second choice is the module split: `common/logger` is a `har`, imported as
`import { Logger } from 'logger'`. It is 36 lines and exists only so error
paths can log without every page reaching for `hilog` with its own domain and
tag. Small, but the right shape for shared plumbing.

## Implementation steps

1. **Own the controller, not the component**: construct a
   `RichEditorController`, wrap it in `RichEditorOptions`, and pass that to
   `RichEditor(...)`. Everything you do to the document goes through the
   controller.
2. **Set the typing style in `onReady`.** Attributes on the `RichEditor` itself
   style the box; `setTypingStyle` is what governs text the user is about to
   type, and it is the style the serialiser will later read back off each span.
3. **Insert images at the caret**: `addImageSpan(uri, { offset:
   controller.getCaretOffset() })`, once per picked URI.
4. **Read the document back with `getSpans()`** - an array of
   `RichEditorTextSpanResult | RichEditorImageSpanResult` in document order.
5. **Discriminate the two by a field**, not by a type test: a text span has
   `value`, an image span does not.
6. **Convert colours.** ArkUI hands back `#AARRGGBB`; CSS wants `#RRGGBBAA`.
   Rotate the alpha from front to back before writing it into the style
   attribute.
7. **Copy each picked image into the app sandbox** and reference it as
   `file://<sandbox path>`. A `photoUris` URI belongs to the gallery and the
   `Web` cannot resolve it.
8. **Enable `fileAccess(true)` on the `Web`**, otherwise the `file://` images
   are blocked and the article renders text-only.
9. **Size the Web to its content** with `layoutMode(WebLayoutMode.FIT_CONTENT)`
   and `constraintSize({ maxHeight: 'auto' })` so it can live inside a `Scroll`
   without a fixed height.
10. **Set `KeyboardAvoidMode.RESIZE`** in the ability so the editor shrinks
    rather than being pushed off-screen.

## Verified snippets

All snippets are from `SimpleArticleEdit.zip`. All are as shipped - no finding
was recorded against this sample.

**The editor and its image insert - `entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
private richEditorController: RichEditorController = new RichEditorController();
private richEditorOptions: RichEditorOptions = { controller: this.richEditorController };

// toolbar item: 上传图片
.onClick(() => {
  fileSelect().then((uri: Array<ResourceStr>) => {
    uri.forEach((item: ResourceStr) => {
      this.richEditorController.addImageSpan(item, {
        offset: this.richEditorController.getCaretOffset(),
      });
    });
  });
});

// the editor
RichEditor(this.richEditorOptions)
  .width('100%')
  .layoutWeight(2)
  .onReady(() => {
    this.richEditorController.setTypingStyle({
      fontWeight: FontWeight.Normal,
      fontColor: Color.Black,
      fontSize: 16,
      fontStyle: FontStyle.Normal,
      fontFamily: 'HarmonyOS Sans',
      letterSpacing: 1,
      lineHeight: 25
    });
  })
  .placeholder($r('app.string.place_hold_string'), {
    fontColor: Color.Gray,
    font: { size: 16, weight: FontWeight.Normal, family: 'HarmonyOS Sans', style: FontStyle.Normal }
  });
```

**`setTypingStyle` inside `onReady` is the load-bearing line.** It is not a
cosmetic default: every text span the user creates is stamped with these
values, `getSpans()` reads them back as `item.textStyle`, and the serialiser
turns them into the CSS that makes the detail page match the editor. Styling
the `RichEditor` component with `.fontSize(16)` instead would style the
container and leave `textStyle` unpopulated, and the published article would
fall back to the serialiser's defaults.

`getCaretOffset()` is re-read inside the `forEach`, so nine picked images
insert in order at the advancing caret rather than all at the original
position. `layoutWeight(2)` inside a `Scroll`'d `Column` gives the body the
remaining height under the 56 vp title field.

**The serialiser - `entry/src/main/ets/pages/SpanToHtml.ets`** (as shipped)

```typescript
export function SpansToHtml(spans: Array<RichEditorImageSpanResult | RichEditorTextSpanResult>,
  context: common.UIAbilityContext): string {
  let html = '<!DOCTYPE html>\n<html>\n<head>\n<meta charset="UTF-8">\n' +
    '<meta name="viewport" content="width=device-width, initial-scale=1"> <!--HTML适配移动端设备-->\n' +
    ' </head>\n<body>\n<div style="width:100%;">\n';
  spans.forEach(span => {
    let spanToHtml = '';
    let item = span as RichEditorTextSpanResult;
    if (item.value !== undefined) {                       // text spans have `value`; image spans do not
      let text: string = item.value;
      if (item.value !== '\n' && item.textStyle !== undefined) {
        let fontColor = argb2Rgba(item.textStyle.fontColor.toString());
        let size =
          `<span style="white-space: pre-wrap;` +
            (item.textStyle.fontSize !== undefined ? `font-size:${item.textStyle.fontSize}px;` : `font-size:20px;`) +
            (item.textStyle.fontColor !== undefined ? `color:${fontColor};` : ``) +
            (item.textStyle.letterSpacing !== undefined ? `letter-spacing:${item.textStyle.letterSpacing};` : ``) +
            (item.textStyle.lineHeight !== undefined ? `line-height:${item.textStyle.lineHeight}px;` :
              `line-height: 25px;`) + `" >`;
        spanToHtml = size + text + '</span>';
      } else {
        spanToHtml = `<span style="white-space: pre-wrap; font-size:20px">${text}</span>`;
      }
    } else {
      let item = span as RichEditorImageSpanResult;
      let imgUrl = item.valueResourceStr as string;
      let fileSandbox = getSandbox(imgUrl, context);      // gallery uri -> sandbox copy
      spanToHtml = `<img src='file://${fileSandbox}' width="100%", height="auto">`;
    }
    html = html + spanToHtml;
  });
  return html + '</div>\n</body>\n</html>';
}

// ArkTS gives #AARRGGBB; CSS wants #RRGGBBAA
function argb2Rgba(argbColor: string): string {
  if (argbColor === undefined || argbColor.length !== 9) {
    return argbColor;
  }
  return '#' + argbColor.substring(3, argbColor.length) + argbColor.substring(1, 3);
}
```

**`white-space: pre-wrap` is what preserves the author's line breaks.**
`RichEditor` reports a paragraph break as a span whose `value` is `'\n'`, and
HTML collapses whitespace by default - without `pre-wrap` every paragraph would
run together. That is also why the `'\n'` case takes the else branch: a bare
newline has no meaningful style to serialise.

**`argb2Rgba` is the conversion nobody expects to need.** `fontColor.toString()`
on an ArkUI colour yields nine characters, `#` plus `AARRGGBB`; CSS hex with
alpha is `#RRGGBBAA`. The function rotates the two alpha digits from the front
to the back, and returns the input unchanged when the length is not 9 (a named
or `rgb()` colour), which keeps it safe as a pass-through.

The `width="100%", height="auto"` on the `<img>` carries a stray comma - HTML
tolerates it as part of the attribute soup, and the images do render full
width, but it is a typo, not a technique.

**Gallery URI to a loadable path - `entry/src/main/ets/utils/FileUtil.ets`** (as shipped)

```typescript
export async function fileSelect(): Promise<Array<string>> {
  let imgUri: Array<string> = [];
  let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
  photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
  photoSelectOptions.maxSelectNumber = 9;
  let photoPicker = new photoAccessHelper.PhotoViewPicker();
  try {
    let photoSelectResult = await photoPicker.select(photoSelectOptions);
    if (photoSelectResult && photoSelectResult.photoUris && photoSelectResult.photoUris.length > 0) {
      imgUri = photoSelectResult.photoUris;
      return imgUri;
    } else {
      return [];
    }
  } catch (error) {
    hilog.error(0x0000, '[FileUtil]', `PhotoViewPicker failed with err: ${error.code}, ${error.message}`);
    return [];
  }
}

// 沙箱uri(file://...) 拷贝到沙箱路径
export function getSandbox(imgUri: string, context: common.UIAbilityContext): string {
  let fileName = new uri.URI(imgUri).getLastSegment();
  let file = fs.openSync(imgUri);
  let fileSandbox = context.cacheDir + '/' + fileName;
  try {
    fs.copyFileSync(file.fd, fileSandbox);
  } finally {
    fs.close(file);
  }
  return fileSandbox;
}
```

**Two different pieces of plumbing, both required.** `PhotoViewPicker` needs no
permission at all - the picker is the grant, and the app receives read access to
exactly the images the user tapped. But that access is scoped to the app
process; a `Web` render engine asked to load a `photoUris` URI will fail. Hence
`getSandbox`: `uri.URI(...).getLastSegment()` for a filename, `fs.copyFileSync`
into `context.cacheDir`, and the returned absolute path is what goes into
`file://` in the `<img>`.

Note the symmetry with `NEWS-22`: there a document picker's URI is copied into
`filesDir` because Reader Kit needs a real path; here a photo picker's URI is
copied into `cacheDir` because the web engine does. Same rule, two kits - a
picker URI is a read handle, never a storage location.

**Rendering the article - `entry/src/main/ets/pages/ArticleDetail.ets`** (as shipped)

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
    .fileAccess(true)                 // without this the file:// images are blocked
    .mixedMode(MixedMode.All)
    .javaScriptAccess(true)
    .zoomAccess(false)
    .layoutMode(WebLayoutMode.FIT_CONTENT)
    .constraintSize({ maxHeight: 'auto' })
    .nestedScroll({
      scrollForward: NestedScrollMode.PARENT_FIRST,
      scrollBackward: NestedScrollMode.SELF_FIRST
    });
}
```

**`onControllerAttached` is the correct hook for `loadData`** - the controller
is only bound to a render process at that point, and calling `loadData` from
`aboutToAppear` would throw. `src: ''` plus a `loadData` in the callback is the
standard "no URL, content is in hand" form; the two `' '` arguments are the base
URL and history URL, which a self-contained document does not need.

Three attributes make the `Web` behave like a block of an ArkUI page rather
than a browser: `FIT_CONTENT` with `maxHeight: 'auto'` lets it grow to its own
content height inside the surrounding `Scroll` (a `Web` is otherwise
viewport-sized and scrolls internally), `zoomAccess(false)` stops pinch-zoom
from fighting the page, and `nestedScroll` with `PARENT_FIRST` forward means
dragging up scrolls the article page, not the embedded document.

## Permissions & config

**None.** The sample declares no `requestPermissions`, and it does not need to:
`PhotoViewPicker` is the permissionless path to gallery images, and everything
after it happens inside the app's own sandbox.

The ability does three things the pages depend on:

```typescript
windowClass.on('keyboardHeightChange', (data) => {
  AppStorage.setOrCreate('keyboardHeight', data);
});
windowClass.getUIContext().setKeyboardAvoidMode(KeyboardAvoidMode.RESIZE);
```

plus the usual `avoidAreaChange` writes into `topRectHeight` and
`bottomRectHeight`. `KeyboardAvoidMode.RESIZE` is the one that matters for an
editor: the default mode offsets the page upward, which pushes the title field
off the top; `RESIZE` shrinks the layout instead, so the toolbar sits on the
keyboard and the caret stays visible.

`route_map.json` declares a single destination, `ArticleDetail`, resolved
through `ArticleDetailBuilder`; `main_pages.json` lists only `pages/Index`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`,
  matching the document. `deviceTypes`: `phone`, `tablet`, `2in1`.
- **Nothing is persisted.** The `Article` lives in `AppStorage` under `'article'`
  and is passed as a navigation parameter; there is one article at a time and it
  is gone on restart. `id` is always `0`, `PUBLISH_DATE` is the constant
  `'2025/09/01 11:00'`, the author is the constant `'小幸运'` and the place
  `'浙江'` - the detail header is a mock around a real body.
- The images are copied into `cacheDir`, which the system may evict. For an
  article that has to survive, copy into `filesDir` instead - the same call with
  a different base directory.
- `getSandbox` re-copies every image on every publish, and never deletes; a
  second publish of the same picture writes it again under the same name.
- `fs.close(file)` in the `finally` is the promise-returning overload, awaited by
  nobody. `fs.closeSync(file)` is the matching call for a synchronous copy.
- Validation is `checkInput`: title non-empty and at least one span. A body of
  pure whitespace passes, and there is no length cap on the body (the title is
  capped at 30 by `maxLength`).
- Publishing navigates from a `setTimeout(..., 1000)` after the toast, so the
  push is not cancellable and fires even if the user backs out in that second.
- `javaScriptAccess(true)` and `mixedMode(MixedMode.All)` are enabled on content
  the app generated itself, which is harmless here but should be turned off, not
  copied, the moment any part of the HTML comes from elsewhere.
- The `Web` is `.width('102%')` - a 2% overhang used to hide the document's own
  side margins. Prefer fixing the margins in the generated CSS.
- The undo, redo and marker toolbar items are toasts (`功能仅展示`, "display
  only"), as are the share/comment/like/star row and the Follow chip on the
  detail page.

## Pitfalls

**No defects were recorded against this document or sample during review.** The
document's snippets are abridged but syntactically valid, the tree matches, and
the constraints match the zip's `compatibleSdkVersion`.

Two review notes that are not findings, but will bite anyone extending it:

- The serialiser handles exactly the styles `setTypingStyle` sets. Add a bold
  button or a colour picker to the toolbar and the published article will drop
  the new attributes silently - `SpansToHtml` must grow with the editor. Text
  decoration, background colour and paragraph alignment (`RichEditorParagraphStyle`)
  are all reachable through the controller and none are serialised.
- The HTML is assembled by string concatenation with no escaping, so a title or
  body containing `<` or `&` is emitted raw into the document. It is the app's
  own content today; it stops being safe the moment a post is rendered from
  another user.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-common-components-richeditor.md` - `RichEditor` usage, typing style, span insertion
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-common-components-richeditor
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-richeditor.md` - `RichEditorController`, `getSpans`, `RichEditorTextSpanResult`, `RichEditorImageSpanResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-richeditor
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `PhotoViewPicker`, `PhotoSelectOptions`, `photoUris`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/02_application-framework/arkts-apis-webview-webviewcontroller.md` - `loadData` and its five arguments
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-webview-webviewcontroller
- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-attributes.md` - `fileAccess`, `layoutMode`, `nestedScroll`, `mixedMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-basic-components-web-attributes
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `copyFileSync`, `openSync`, `close`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `huawei_industry_tree/11_news_reading/docs/24_auto_flip_read.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/auto_flip_read-0000002384511241
- `NEWS-22` - the same picker-URI-to-sandbox rule, with `DocumentViewPicker` and Reader Kit
