---
id: SOCIAL-24
title: Long-press a chat image to read its QR code - detectBarcode over a PixelMap, half-modal action sheet
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/24_long_press_scan_code.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/long_press_scan_code-0000002297010246
sample: huawei_industry_tree/14_social_communication/downloads/LongPressScanCode.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.ArkWeb", "@kit.BasicServicesKit", "@kit.ImageKit", "@kit.PerformanceAnalysisKit", "@kit.ScanKit"]
apis: [detectBarcode, "detectBarcode.decodeImage", "detectBarcode.ByteImage", "detectBarcode.ImageFormat", scanBarcode, scanCore, LongPressGesture, bindSheet, "image.createImageSource", createPixelMap, convertPixelFormat, readPixelsToBuffer, getPixelBytesNumber, Navigation, NavPathStack, NavDestination, webview, window]
permissions: [ohos.permission.INTERNET]
min_api: 20
modules: [entry (entry)]
findings: [HW-14-0057, HW-14-0058, HW-14-0087, HW-14-0088]
status: verified-with-fixes
---

## When to use

Load this card when a **picture already on screen may contain a machine-readable
code and the user should be able to act on it without leaving the app**. The
pattern: no camera, no scan page - the image bytes you already hold are pushed
through `detectBarcode.decodeImage`, and a half-modal sheet raised by a long
press offers the resulting action.

This is the chat variant (a QR code someone messaged you), but the shape is the
same for a saved poster in a gallery viewer, a screenshot in a support thread,
or a product photo in a marketplace listing. The distinguishing constraint is
that the decode runs on a `PixelMap` you produce yourself, so *you* own the
pixel format contract with the scanner - which is exactly where this sample
gets it wrong (`HW-14-0057`).

The interaction half is worth copying even if you decode differently: a
`LongPressGesture` that only sets a boolean, and a `bindSheet` that reads it
with `$$` two-way binding, so the sheet's own dismissal writes the flag back
and the gesture stays re-armable.

## Feature checklist

- A chat page with a QR code image inline in a message bubble.
- Tapping the image opens a full-screen enlarged view over a black backdrop.
- Long-pressing the enlarged image raises a half-modal sheet from the bottom.
- The sheet carries a single QR-code icon with the label 识别当前二维码
  (recognize this QR code).
- Tapping it decodes the code and pushes a `NavDestination` web page onto the
  stack with the decoded URL.
- The web page shows the page title and the URL, and back navigates inside the
  Web history before popping.
- A failed recognition leaves the user on the chat page with a toast rather
  than an empty web view.

## Architecture

One `entry` module, two pages, no model or utility layer. The whole feature is
276 lines of `ChatPage.ets` plus a routed web page.

```
entry/src/main/ets
├── entryability/EntryAbility.ets   full-screen layout, avoid areas -> AppStorage
└── pages
    ├── ChatPage.ets                @Entry: chat UI, enlarged image, sheet, decode
    └── WebPage.ets                 NavDestination + webPageBuilder, opened by route map
entry/src/main/resources/base/profile
├── main_pages.json                 pages/ChatPage only
└── route_map.json                  WebPage -> webPageBuilder
```

The documented tree matches the zip; the doc simply omits the two profile
files, which are the interesting part.

**The design decision worth copying** is that the web page is reached through
the **route map**, not through `main_pages.json`. `ChatPage` holds a single
`@Provide('PageStack') pageInfos: NavPathStack` and pushes by name:

```typescript
this.pageInfos.pushPathByName('WebPage', this.content);
```

`WebPage` is never registered as a router page. It exports a `@Builder`
function, `route_map.json` maps the name `WebPage` to that builder, and
`module.json5` points `routerMap` at the profile. Nothing in `ChatPage` imports
`WebPage`, so the web page and everything ArkWeb drags in are loaded only when
a code is actually decoded. The `@Consume('PageStack')` on the other side
closes the loop without prop drilling.

## Implementation steps

1. **Render the message image twice**: once inline in the bubble, once as a
   full-bleed `enlargedImage` builder gated by a `@State visible: Visibility`.
   The enlarged copy sets `.draggable(false)` so the long press is not stolen
   by the drag framework.
2. **Attach `LongPressGesture({ repeat: true })`** and set the sheet flag from
   both `onAction` and `onActionEnd` - with `repeat: true`, `onAction` fires
   while the finger is still down, so the sheet appears at the press rather
   than at the release.
3. **Bind the sheet with `$$this.isShow`**, not `this.isShow`. The two-way
   binding is what lets a swipe-down dismissal write `false` back into the
   state; a one-way bind leaves the flag stuck true and the second long press
   does nothing.
4. **Build the PixelMap from the resource** with
   `resourceManager.getMediaContent(...)` → `image.createImageSource` →
   `createPixelMap`. In a real chat, substitute the received file's URI.
5. **Convert the pixel format and `await` it** before reading the buffer, and
   declare to `detectBarcode` the same format you converted to (`HW-14-0057`).
6. **Size the buffer from `getPixelBytesNumber()`**, and take `width`/`height`
   from `getImageInfo()` - the three must describe one and the same buffer.
7. **Check `scanResults.length` before indexing** and release the `PixelMap`
   and `ImageSource` in a `finally` (`HW-14-0058`).
8. **Push the decoded value as the route parameter**, and read it in
   `NavDestination.onReady` from `context.pathInfo.param`.

## Verified snippets

All snippets are from `LongPressScanCode.zip`. Corrected forms are marked.

**Long press to half-modal - `entry/src/main/ets/pages/ChatPage.ets`** (as shipped)

```typescript
@Builder
enlargedImage() {
  Text()
    .width('100%')
    .height('100%')
    .backgroundColor(Color.Black)
    .visibility(this.visible);
  Image($r('app.media.qr_code'))
    .width('92%')
    .height(328)
    .draggable(false)
    .visibility(this.visible)
    .margin({ top: 232, left: 16 })
    .onClick(() => {
      this.visible = Visibility.None;
    })
    .gesture(
      LongPressGesture({ repeat: true })
        .onAction(() => {
          this.isShow = true;
        })// 长按动作一结束触发
        .onActionEnd(() => {
          this.isShow = true;
        })
    )
    .bindSheet($$this.isShow, this.popupBuilder(), {
      height: 176,
      backgroundColor: 'rgba(255, 255, 255, 0.9)',
    });
}
```

**Three details carry this.** `draggable(false)` is mandatory: `Image` is
draggable by default, and a long press on a draggable image starts a drag
instead of firing the gesture. `$$this.isShow` makes the sheet's state flow
back - `bindSheet` closes itself on swipe-down or backdrop tap, and without the
two-way sigil the boolean would stay `true` forever. And `height: 176` with a
`Column` sized `'100%'` inside means the sheet is a fixed-height action strip,
not a resizable panel; no detents are declared, so the sheet has exactly one
resting position.

The black `Text()` with no content is the backdrop - a cheap full-screen scrim
that shares the same `visibility` binding as the image, so one state field
raises both.

**The decode - same file** (corrected, see `HW-14-0057`)

```typescript
async decodeImageBuffer(pixelMap: image.PixelMap) {
  await pixelMap.convertPixelFormat(image.PixelMapFormat.NV21);   // FIX: awaited, and NV21 to match the declaration below
  let info = await pixelMap.getImageInfo();
  let pixelBytesNumber: number = pixelMap.getPixelBytesNumber();
  let buffer = new ArrayBuffer(pixelBytesNumber);
  await pixelMap.readPixelsToBuffer(buffer);
  let byteImg: detectBarcode.ByteImage = {
    byteBuffer: buffer,
    width: info.size.width,
    height: info.size.height,
    format: detectBarcode.ImageFormat.NV21
  };
  let options: scanBarcode.ScanOptions =
    { scanTypes: [scanCore.ScanType.ALL], enableMultiMode: true, enableAlbum: false };
  try {
    return detectBarcode.decodeImage(byteImg, options).then((result: detectBarcode.DetectResult) => {
      return result;
    }).catch((error: BusinessError) => {
      return 'code' + error.code + 'message' + error.message;
    });
  } catch (error) {
    return 'code' + error.code + 'message' + error.message;
  }
}
```

**`ByteImage` is a raw contract, and the sample breaks both halves of it.** The
shipped code calls `convertPixelFormat(image.PixelMapFormat.NV12)` without
`await` and then declares `detectBarcode.ImageFormat.NV21`. Un-awaited, the
following `getPixelBytesNumber()` and `readPixelsToBuffer()` race a conversion
that is still running, so the buffer may still be RGBA-sized - and even once
the conversion lands, NV12 and NV21 differ in the interleaving order of the
chroma plane, so the decoder reads U where V is. Nothing throws; recognition
just fails on some runs and works on others. Pick one format and use it on both
lines. NV21 is the safer pick here because it is what the scan API expects for
single-plane YUV input, and it is what the declaration already says.

Note the deliberately odd return type: `DetectResult | string`. The caller
distinguishes the two with `typeof result !== 'string'`. It works, but a
`{ ok, result, error }` shape or a rethrown `BusinessError` would be honest
about which branch is the failure.

**The call site, with the empty-result branch - same file** (corrected, see `HW-14-0058`)

```typescript
async scanQRFromImage() {
  let context = this.getUIContext().getHostContext() as Context;
  let resourceMgr = context.resourceManager;
  let array = await resourceMgr.getMediaContent($r('app.media.qr_code').id);
  let imageSource = image.createImageSource(array.buffer);
  let pixelMap = await imageSource.createPixelMap();
  try {
    let result = await this.decodeImageBuffer(pixelMap);
    if (typeof result !== 'string' && result.scanResults.length > 0) {   // FIX: length checked
      this.content = result.scanResults[0].originalValue;
      this.pageInfos.pushPathByName('WebPage', this.content);
    } else {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.recognition_failed') });
    }
    this.isShow = false;
  } catch (error) {
    const err = error as BusinessError;
    this.getUIContext().getPromptAction().showToast({ message: err.message });
  } finally {
    pixelMap.release();          // FIX: shipped code releases only on the success path
    imageSource.release();
  }
}
```

**The shipped version has one path with no cleanup and no user-facing story.**
On an image with no barcode, `decodeImage` resolves with an empty
`scanResults`, `scanResults[0].originalValue` throws a `TypeError`, and control
jumps into the catch - which toasts `err.message`, an internal string, and
skips the two `release()` calls and the `isShow = false` below them. So every
unsuccessful attempt leaks a native `PixelMap` and an `ImageSource`, leaves the
sheet open, and shows the user a stack-flavoured message. Guarding on
`length > 0` routes the empty case into the existing 识别失败 (recognition
failed) toast, and moving the releases into `finally` makes both paths pay for
their own cleanup.

`getMediaContent(...).buffer` is worth noting: the sample decodes a bundled
resource, so this is a demo shortcut. For a received message, build the source
from the file URI instead - `image.createImageSource(uri)` - and everything
below stays identical.

**The routed web page - `entry/src/main/ets/pages/WebPage.ets`** (as shipped)

```typescript
@Builder
export function webPageBuilder() {
  WebPage();
}

@Component
struct WebPage {
  @Consume('PageStack') pageInfos: NavPathStack;
  private controller: WebviewController = new webview.WebviewController();
  @State url: string = '';

  build() {
    NavDestination() {
      this.titleBarBuilder();
      Web({ src: this.url, controller: this.controller })
        .layoutWeight(1)
        .zoomAccess(false)
        .javaScriptAccess(true)
        .onTitleReceive((event) => {
          this.title = event.title;
        })
        .expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.BOTTOM]);
    }
    .onReady((context) => {
      this.url = context.pathInfo.param as string;
    })
    .onBackPressed(() => {
      if (this.controller.accessBackward()) {
        this.controller.backward();
        return true;
      } else {
        return false;
      }
    });
  }
}
```

**`onBackPressed` returning a boolean is the whole back-navigation policy.**
Return `true` and the event is consumed - here, to walk the Web component's own
history. Return `false` and `Navigation` pops the destination. So one predicate,
`accessBackward()`, gives the user "back inside the page until there is nowhere
left, then back out of the page", which is what a browser-shaped destination
should do.

`onReady` rather than `aboutToAppear` is the right place to read the parameter:
`pathInfo` is only attached when the destination is mounted into the stack.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" }
],
"pages": "$profile:main_pages",
"routerMap": "$profile:route_map"
```

- `INTERNET` is `system_grant`, so no `reason` or `usedScene` is required and
  nothing is prompted at runtime.
- `detectBarcode` needs **no** permission: the pixels are already in the app's
  process. Only the follow-on web page needs the network.
- `route_map.json` is what makes `pushPathByName('WebPage', ...)` resolve.
  Forgetting it is a silent no-op push, not a build error.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- The decoded QR code is a **bundled resource**, not a received message. There
  is no message list, no image download, and the "chat" is two static bubbles.
- `enableMultiMode: true` is passed but only `scanResults[0]` is ever read, so
  a picture with several codes silently loses all but the first.
- `enableAlbum: false` is correct for this flow - the album picker belongs to
  the interactive `scanBarcode` UI, not to `detectBarcode`.
- The sheet has one fixed `height: 176` and no detents, so it cannot be dragged
  taller; adding actions means changing the constant.
- `EntryAbility` registers `avoidAreaChange` and never removes it, and drops
  the promise from `setWindowLayoutFullScreen` - the same boilerplate defect
  found across this industry's samples.

## Pitfalls

- **`HW-14-0057`** (B/medium, confirmed): `convertPixelFormat(NV12)` is called
  without `await` while the `ByteImage` declares `ImageFormat.NV21`, so the
  decoder receives a buffer matching neither the declared size nor the declared
  chroma order and recognition fails nondeterministically behind a generic
  toast. Fix: `await` the conversion and use one format on both lines.
- **`HW-14-0058`** (B/low, probable): the no-code-detected path indexes
  `scanResults[0]` unchecked, so a `TypeError` jumps to the catch, skipping
  `pixelMap.release()`, `imageSource.release()` and `isShow = false`, leaking
  native objects and showing a raw internal message. Fix: check
  `scanResults.length` and release in a `finally`.

## References

- `documentation/harmonyos-references/04_media/scan-imagedecode.md` - `detectBarcode.decodeImage`, `ByteImage`, `ImageFormat`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/scan-imagedecode
- `documentation/harmonyos-references/04_media/arkts-apis-image-pixelmap.md` - `convertPixelFormat`, `readPixelsToBuffer`, `getPixelBytesNumber`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-pixelmap
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-longpressgesture.md` - `repeat`, `onAction` vs `onActionEnd`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-longpressgesture
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - `bindSheet` and `SheetOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavPathStack`, `pushPathByName`, route maps
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-web.md` - `Web`, `onTitleReceive`, `accessBackward`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-web
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `ohos.permission.INTERNET`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `SOCIAL-22` - the same "chat message links into a routed Web destination" flow, with history kept in a float bubble
