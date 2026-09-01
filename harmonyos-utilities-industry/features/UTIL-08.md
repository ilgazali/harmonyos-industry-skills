---
id: UTIL-08
title: Web page poster - full-page webPageSnapshot, packToFile, then the system share sheet
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/08_web_poster_produce_and_share.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/web_poster_produce_and_share-0000002282232164
sample: huawei_industry_tree/15_utilities/downloads/网页海报生成及分享示例代码.zip
kits: ["@kit.AbilityKit", "@kit.ArkData", "@kit.ArkUI", "@kit.ArkWeb", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.PerformanceAnalysisKit", "@kit.ShareKit"]
apis: [Web, "webview.WebviewController", enableWholeWebPageDrawing, webPageSnapshot, "image.createImagePacker", ImagePacker, packToFile, PackingOption, "ImagePacker.release", "fileIo.openSync", "fileIo.accessSync", "fileIo.close", "fileIo.unlinkSync", "fileUri.getUriFromPath", "systemShare.SharedData", "systemShare.ShareController", SharePreviewMode, SelectionMode, "utd.UniformDataType", "resourceManager.getStringSync"]
permissions: [ohos.permission.INTERNET]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0026, HW-15-0101, HW-15-0102]
status: verified-with-fixes
---

## When to use

Load this card when the app must **turn on-screen content into an image file
and hand it to the system share sheet** - "share this page as a picture",
which is how articles, receipts, itineraries, scores and product pages get
posted to chat apps. The sample's source is a `Web` component, but the second
half of the pipeline works on any `PixelMap`, including one from
`componentSnapshot` over your own ArkUI subtree.

The chain has four links and each one has a rule. `enableWholeWebPageDrawing`
must be called **before the `Web` component is created**, or the snapshot is
limited to the visible viewport. `webPageSnapshot` returns a `PixelMap`, not a
file. `systemShare` takes a **URI**, so the `PixelMap` has to be encoded to a
real sandbox file with `ImagePacker.packToFile` and converted with
`fileUri.getUriFromPath`. And the packer holds native memory, so `release()`
belongs in a `finally`.

**Read `HW-15-0026` before adopting the share function.** Its error handling
is arranged so that a failed encode still opens the share sheet, over a
zero-byte file.

## Feature checklist

- A full-page `Web` component loading a URL taken from a string resource.
- A button above it labelled 生成海报 (generate poster).
- Pressing it captures the **entire** page - not just the visible viewport -
  up to a 16000 x 16000 px bound.
- The capture is encoded to JPEG at quality 100 and written to
  `filesDir/webPic.jpeg`, replacing any previous capture.
- The system share sheet opens with the image as a detail-mode preview and
  single-target selection.
- The `Web` runs with JavaScript, DOM storage, local file, image and zoom
  access on, and geolocation and mixed content off.

## Architecture

One `entry` module, four files - the smallest sample in this industry.

```
entry/src/main/ets
├── entryability/EntryAbility.ets       standard, loads pages/WebPage
├── entrybackupability/EntryBackupAbility.ets
├── pages/WebPage.ets                   the Web component, the button, the snapshot call
└── toolability/ShareAbility.ets        shareFunc(context, pixelMap) - pack + share
```

The documented tree matches the zip.

**The design decision worth copying** is that `shareFunc` is a free exported
function taking `(context, imagePixelMap)` rather than a method on the page or
a class with state:

```typescript
export async function shareFunc(context: common.UIAbilityContext, imagePixelMap: image.PixelMap)
```

Everything it needs arrives as an argument, so it is reusable against any
`PixelMap` from any producer - a `Web` snapshot today, a `componentSnapshot`
or a canvas tomorrow - and it is testable without a UI. That boundary is the
right one: the page's job is to produce pixels, the tool's job is to turn
pixels into a share.

What the sample gets wrong is that the boundary is one-way. `shareFunc`
reports failure only to hilog and returns nothing, so the caller cannot know
whether the poster was produced. Make it `Promise<boolean>` or let it throw,
and the caller can show the user something.

## Implementation steps

1. **Call `webview.WebviewController.enableWholeWebPageDrawing()` in
   `aboutToAppear`,** before the `Web` component exists. It is a static, and
   it configures the drawing mode the component will be created with.
2. **Give the `Web` a `WebviewController`** and keep it as a field; the
   snapshot is taken through the controller, not the component.
3. **Call `webPageSnapshot({ id, size }, callback)`** with a size large enough
   to hold the whole document. Check `error` first, then that
   `result.imagePixelMap` exists.
4. **Pass the `PixelMap` to the share tool** with the UIAbility context.
5. **Delete any previous capture with `unlinkSync`, not `rmdirSync`**
   (`HW-15-0026`).
6. **Open the destination with `WRITE_ONLY | CREATE | TRUNC`** so a shorter
   image cannot leave a tail of the previous one.
7. **`await packToFile`, and close the fd and `release()` the packer in a
   `finally`.**
8. **Return on a pack failure instead of sharing an empty file**
   (`HW-15-0026`).
9. **`await` the share call at the call site and attach a `catch`**
   (`HW-15-0026`).

## Verified snippets

All snippets are from `网页海报生成及分享示例代码.zip`
(`WebPosterProduceAndShare/`). Corrected forms are marked.

**Enabling full-page drawing and taking the snapshot — `entry/src/main/ets/pages/WebPage.ets`** (corrected, see `HW-15-0026`)

```typescript
private webController: webview.WebviewController = new webview.WebviewController();
@State url: string =
  (this.getUIContext().getHostContext() as Context).resourceManager.getStringSync($r('app.string.default_url').id);

aboutToAppear(): void {
  try {
    webview.WebviewController.enableWholeWebPageDrawing(); // enable whole-page drawing
  } catch (error) {
    hilog.error(0x0000, 'testTag',
      `ErrorCode: ${(error as BusinessError).code}, Message: ${(error as BusinessError).message}`);
  }
}

// on the button:
.onClick(() => {
  try {
    this.webController.webPageSnapshot(
      { id: '1234', size: { width: '16000px', height: '16000px' } },
      (error, result) => {
        if (error) {
          hilog.error(0x0000, 'testTag',
            `ErrorCode: ${(error as BusinessError).code}, Message: ${(error as BusinessError).message}`);
          return;
        }
        if (result && result.imagePixelMap) {
          let context = this.getUIContext().getHostContext() as common.UIAbilityContext;
          if (context) {
            shareFunc(context, result.imagePixelMap)          // FIX: shipped code fires this bare
              .catch((err: BusinessError) => {
                hilog.error(0x0000, 'testTag', `share failed. Code: ${err.code}, message: ${err.message}`);
              });
          }
        }
      });
  } catch (error) {
    hilog.error(0x0000, 'testTag',
      `ErrorCode: ${(error as BusinessError).code},  Message: ${(error as BusinessError).message}`);
  }
});
```

**`enableWholeWebPageDrawing` is a static called before the component exists,
and that ordering is the whole trick.** A `Web` component normally rasterises
only what is on screen; the flag tells the engine to keep the full document
surface so `webPageSnapshot` has something to copy beyond the viewport. Called
after the component is up, it does not retroactively apply - hence
`aboutToAppear` rather than `onPageEnd` or the button handler.

The `size` of `16000px` in each axis is a ceiling, not a target: it bounds how
much of the document the snapshot may cover. It is also the reason to treat
this as expensive - a tall page at that bound is a very large `PixelMap`, and
it lives in native memory until it is packed and dropped.

`shareFunc` is `async` and the shipped code invokes it with neither `await`
nor `.catch`. The two synchronous `fileIo` calls at the top of it -
`accessSync` and `openSync` - can throw, and inside an async function a throw
becomes a rejection; with no handler it is an unobserved rejection and the
user sees the button do nothing at all.

**Staging and packing — `entry/src/main/ets/toolability/ShareAbility.ets`** (corrected, see `HW-15-0026`)

```typescript
export async function shareFunc(context: common.UIAbilityContext, imagePixelMap: image.PixelMap) {
  let dir: string = context.filesDir;
  let filePath: string = dir + '/webPic.jpeg';
  let res = fileIo.accessSync(filePath);
  if (res) {
    try {
      fileIo.unlinkSync(filePath);        // FIX: shipped code calls rmdirSync on a regular file
      hilog.info(0x0000, 'testTag', 'rm file success');
    } catch (err) {
      hilog.error(0x0000, 'testTag', 'unlink failed with error message: ' + err.message + ', code: ' + err.code);
    }
  }
  let fileSand =
    fileIo.openSync(filePath, fileIo.OpenMode.WRITE_ONLY | fileIo.OpenMode.CREATE | fileIo.OpenMode.TRUNC);

  const imagePackerApi: image.ImagePacker = image.createImagePacker();
  let packOpts: image.PackingOption = { format: 'image/jpeg', quality: 100 };
  let packed = false;                     // FIX: the shipped code has no success flag
  try {
    await imagePackerApi.packToFile(imagePixelMap, fileSand.fd, packOpts);
    packed = true;
  } catch (error) {
    hilog.error(0x0000, 'testTag',
      `Failed to pack the image to file.code ${error.code}, message is ${error.message}`);
  } finally {
    fileIo.close(fileSand.fd);            // close the handle explicitly
    imagePackerApi.release();             // release the packer's native memory
  }
  if (!packed) {
    return;                               // FIX: shipped code falls through and shares an empty file
  }
  // ... share
}
```

**`packToFile` writes into a file descriptor you own, which is why the open
mode matters.** `WRITE_ONLY | CREATE | TRUNC` creates the file if it is
missing and truncates it to zero if it is not, so a second, smaller poster
cannot leave the tail of the first one behind. The `finally` is doing two
different jobs: `fileIo.close` returns the descriptor, and
`imagePackerApi.release()` frees the encoder's native buffers, which are not
reclaimed by the ArkTS GC.

The `TRUNC` is also what makes the missing guard dangerous. The file is
emptied *before* the encode is attempted, so if `packToFile` throws, what is
on disk is a zero-byte `webPic.jpeg` - and the shipped code's `catch` only
logs, then falls straight through to `ShareController.show`. The user gets a
share sheet over an empty image and no indication anything failed. One boolean
and one early return close it.

The `rmdirSync` on the line above is the same class of mistake in reverse:
`filePath` is a regular file, so `rmdirSync` always fails with `ENOTDIR`. It
happens to be harmless here because the `TRUNC` open does the real clearing,
which is why nobody noticed - and the document reproduces the line verbatim,
so readers will copy it.

**The share — same file** (as shipped)

```typescript
let shareData: systemShare.SharedData = new systemShare.SharedData({
  utd: utd.UniformDataType.JPEG,
  uri: fileUri.getUriFromPath(filePath),
  description: 'webPic',
  label: '图片',                                       // the sheet's title: "image"
});
let shareController: systemShare.ShareController = new systemShare.ShareController(shareData);
shareController.show(context, {
  previewMode: systemShare.SharePreviewMode.DETAIL,
  selectionMode: systemShare.SelectionMode.SINGLE
}).then(() => {
  hilog.info(0x0000, 'testTag', 'HuaweiShare_show');
}).catch((error: BusinessError) => {
  hilog.error(0x0000, 'testTag', `HuaweiShare_show error. Code: ${error.code}, message: ${error.message}`);
});
```

**Three fields decide what the sheet can do.** `utd` is the uniform type
descriptor - `UniformDataType.JPEG` here - and it is what the system matches
against each target app's declared accepted types, so getting it wrong
produces an empty or wrong list of targets rather than an error. `uri` must be
a URI, which is why `fileUri.getUriFromPath` wraps the sandbox path; a bare
`/data/.../webPic.jpeg` string is not shareable. `label` and `description` are
what the sheet displays above the preview.

`SharePreviewMode.DETAIL` shows the image itself as the preview rather than a
generic file row, which is the point of a poster. `SelectionMode.SINGLE`
limits the user to one target per share - use `BATCH` if the flow should allow
several.

Sharing by URI is also why this feature needs no media or storage permission:
the system grants the receiving app transient read access to that one file.

## Permissions & config

```json5
"requestPermissions": [
  { "name": "ohos.permission.INTERNET" },
]
```

`INTERNET` is `system_grant`, needed only by the `Web` component to load the
page - the snapshot, the packing and the share need nothing. `deviceTypes` is
`["phone", "tablet", "2in1"]`.

The `Web` component's access flags are the security surface worth reviewing
before reuse:

```typescript
Web({ src: this.url, controller: this.webController })
  .domStorageAccess(true)
  .javaScriptAccess(true)
  .fileAccess(true)          // local file access - tighten this for untrusted URLs
  .imageAccess(true)
  .onlineImageAccess(true)
  .zoomAccess(true)
  .geolocationAccess(false)
  .mixedMode(MixedMode.None) // no HTTP subresources on an HTTPS page
```

`geolocationAccess(false)` and `mixedMode(MixedMode.None)` are the two good
defaults here. `fileAccess(true)` is the one to reconsider: it lets page script
reach local files, which is only appropriate for content you control.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **`enableWholeWebPageDrawing` must precede the `Web` component's creation.**
  Called later it has no effect on an already-created component.
- The snapshot bound is 16000 x 16000 px. A longer document is truncated, and
  a page anywhere near that bound produces a very large `PixelMap` - the
  memory cost is real and the packer must be released.
- `quality: 100` JPEG is a large file for a full-page capture. Lower it if the
  share target has a size limit.
- Only one poster path is used, `filesDir/webPic.jpeg`, so a second capture
  always overwrites the first. Concurrent captures would race on the same fd.
- The URL comes from `app.string.default_url` read at construction with
  `getStringSync`; there is no address bar and no navigation controls, so the
  sample captures whatever that resource points at (plus wherever the user
  navigates from inside the page).
- `shareFunc` returns `Promise<void>` and swallows every failure into hilog,
  so no caller can present an error state.

## Pitfalls

- **`HW-15-0026`** (B/medium, confirmed): the share flow's error handling is
  broken in three places. `ShareAbility.ets` catches a `packToFile` failure,
  logs it, and falls through to `systemShare` with the `TRUNC`-emptied
  `webPic.jpeg`, so the user can share a zero-byte image. The cleanup calls
  `rmdirSync(filePath)` on a regular file, which always throws `ENOTDIR` -
  a dead path the document reproduces verbatim. And `WebPage.ets` fires the
  `async shareFunc` with neither `await` nor `.catch`, so an `openSync` or
  `accessSync` failure becomes an unobserved rejection and the button appears
  to do nothing. Fix: guard the share on a pack-success flag, use
  `unlinkSync`, and `await` + `catch` at the call site.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web.md` - the `Web` component and its access attributes
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-basic-components-web
- `documentation/harmonyos-references/02_application-framework/arkts-apis-webview-WebviewController.md` - `enableWholeWebPageDrawing`, `webPageSnapshot`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-webview-webviewcontroller
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagepacker.md` - `createImagePacker`, `packToFile`, `PackingOption`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-references/06_application-services/share-system-share.md` - `SharedData`, `ShareController.show`, `SharePreviewMode`, `SelectionMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/share-system-share
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `openSync` modes, `accessSync`, `unlinkSync`, `close`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-guides/04_system/permissions-for-all.md` - `INTERNET` is system_grant
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/permissions-for-all
- `UTIL-06` - the other sample here that stages a file into the sandbox before handing its path to a system service
