---
id: UTIL-27
title: Save a Base64 image out of an H5 page - javaScriptProxy in, onContextMenuShow out
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/27_base64_image_save_h5.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/base64_image_save_h5-0000002334961556
sample: huawei_industry_tree/15_utilities/downloads/Base64ImageSaveH5.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.ArkWeb", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [Web, javaScriptProxy, onContextMenuShow, ContextMenuMediaType, WebContextMenuParam, "webview.WebviewController", "util.Base64Helper", encodeToStringSync, decode, "util.Type", "photoAccessHelper.getPhotoAccessHelper", createAsset, "fs.open", "fs.write", "fs.close", "abilityAccessCtrl.requestPermissionsFromUser", "@CustomDialog", getRawFileContentSync]
permissions: [ohos.permission.WRITE_IMAGEVIDEO]
min_api: 20
modules: [entry]
findings: [HW-15-0062, HW-15-0063, HW-15-0016, HW-15-0100, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when app-owned images are rendered **inside a `Web` component as
Base64 data URLs** and the user must be able to long-press one and save it to
the gallery. That is the common shape for hybrid apps whose catalogue,
marketing page or report is authored in HTML but whose assets ship in the HAP:
no network fetch, no file URLs, no `fileAccess` - the bytes go from
`rawfile` through a JS bridge into an `<img src="data:image/png;base64,…">`.

Two directions of traffic make the feature, and they use two different
mechanisms. **Into** the page: `javaScriptProxy` injects an ArkTS object under
a global name (`window.picture`) whose listed methods H5 may call, and the page
calls `picture.chosePicture()` on load to receive the array of Base64 strings.
**Out of** the page: `onContextMenuShow` intercepts the long-press, and when the
target is an image, `event.param.getSourceUrl()` hands the app the very data URL
it injected. The app decodes it and writes it to the album.

**Read `HW-15-0062` before adopting any of the save half.** The round trip is
encoded in MIME mode and decoded in BASIC mode, and the file write is not
awaited, so the shipped save path fails for real images while telling the user
it succeeded. The `Web` half is sound; the album half needs the corrections
below.

## Feature checklist

- The app opens on a full-height `Web` showing a local H5 gallery of fifteen
  images with Chinese captions.
- On load the page calls into ArkTS, receives fifteen Base64 strings and sets
  them as `data:image/png;base64,…` sources.
- Long-pressing any image suppresses the default Web context menu and opens a
  bottom sheet with 保存图片 (save image), a save button, a cancel button and a
  close cross.
- Save requests the album write permission, decodes the Base64 payload and
  writes it into the system gallery as a PNG.
- A toast reports the outcome.
- Tapping outside the sheet or pressing back dismisses it.

## Architecture

One `entry` module. The H5 side is two rawfiles; the ArkTS side is a page, a
dialog and three utility files.

```
entry/src/main/ets
├── common/Constants.ets            IMAGE_DATA: the fifteen rawfile names
├── component/SavePopUp.ets         @CustomDialog - the save sheet, orchestrates permission + decode + write
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── pages/Photos.ets                @Entry - the Web, the proxy object, onContextMenuShow
└── utils
    ├── Base64Utils.ets             base64ToArrayBuffer
    ├── Logger.ets
    ├── PhotoUtils.ets              saveImageToAlbum - createAsset + fs.write
    └── UserPermissionUtils.ets     reqPermissionsFromUser
entry/src/main/resources/rawfile
├── Photos.html                     the gallery, calls picture.chosePicture() on body load
└── image0..14.png                  the assets, never referenced by URL
```

The documented tree matches the zip.

**The design decision worth avoiding** is how the three utility files are
packaged. `PhotoUtils` and `UserPermissionUtils` are declared `@Component
struct` with an empty `build() {}` and then exported as
`export const photoUtils = new PhotoUtils()`. They are never placed in any
component tree, so they are not components in any meaningful sense - but their
methods call `this.getUIContext().getPromptAction()`, which requires a live UI
instance behind `this`. Every one of those calls sits on a failure path (a
refused permission, a failed write), so the code that reports the problem is
itself the code that cannot run (`HW-15-0063`). If you want a utility, write a
plain class or a bare function and **pass the `UIContext` in as a parameter**.
`SavePopUp` already does exactly that for its own context, so the correct
plumbing is one line away.

`Photos.ets` also declares `PictureManagement` as an `@Component struct` and
`new`s it as the proxy object. That one is closer to defensible - the proxy
object needs to be a plain object with methods and it happens to work - but
`chosePicture()` calls `this.getUIContext().getHostContext()` on the same
never-mounted struct, and the correct form is a plain class holding a context
handed in by the page.

## Implementation steps

1. **Ship the images as rawfiles** and list their names in a constant; do not
   reference them from the HTML by path (`fileAccess(false)` is set, and
   deliberately).
2. **Read each with `getRawFileContentSync` and encode with
   `base64helper.encodeToStringSync(arrayBuff, util.Type.MIME)`.** MIME mode
   inserts CRLF every 76 characters, which is legal in a data URL and what
   makes the string safe to embed - but it binds you to a MIME-mode decoder on
   the way back (`HW-15-0062`).
3. **Expose the encoder over `javaScriptProxy`** with an explicit `methodList`.
   Only listed methods are reachable from JS; anything else on the object stays
   private.
4. **Have the page call the bridge on load** and set each `<img>`'s `src` to
   `'data:image/png;base64,' + payload`.
5. **Lock the Web down**: `geolocationAccess(false)`, `fileAccess(false)`. The
   page needs neither.
6. **Intercept the long press with `onContextMenuShow`**, check
   `event.param.getMediaType() === ContextMenuMediaType.Image`, take the part of
   `getSourceUrl()` after the comma, and **return `true`** to suppress the
   built-in menu. Returning `false` lets the default menu through.
7. **Restore the CRLFs** that the Web layer percent-encoded
   (`%0D%0A` → `\r\n`) - the sample does this, and it is only correct because
   the payload is MIME.
8. **Decode in the same mode you encoded in**: `decode(str, util.Type.MIME)`
   (`HW-15-0062`).
9. **`await` the write before closing the fd, and guard the close**
   (`HW-15-0062`, `HW-15-0016`).
10. **Toast the real outcome**, not a fixed success (`HW-15-0062`).
11. **Prefer `showAssetsCreationDialog` or a `SaveButton` over
    `WRITE_IMAGEVIDEO`** - the declared permission is restricted and will be
    auto-denied on an ordinarily signed build (`HW-15-0063`).

## Verified snippets

All snippets are from `Base64ImageSaveH5.zip`. Corrected forms are marked.

**The bridge, both directions — `entry/src/main/ets/pages/Photos.ets`** (as shipped)

```typescript
Web({ src: $rawfile('Photos.html'), controller: this.webController })
  .javaScriptProxy({
    // 配置ArkTS与H5的双向通信能力
    object: this.pictureManagement,   // 注入ArkTS侧的对象实例
    name: 'picture',                  // H5侧调用时使用的对象名称（window.picture）
    methodList: ['chosePicture'],     // 暴露给H5调用的方法列表
    controller: this.webController    // 绑定当前Web组件的控制器
  })
  .height('100%')
  .geolocationAccess(false)
  .fileAccess(false)
  .onContextMenuShow((event) => {
    // 检测到长按图片，弹出菜单
    if (event.param.getMediaType() === ContextMenuMediaType.Image) {
      // 获取逗号后的实际数据
      this.base64Image = event.param.getSourceUrl().split(',')[1];
      this.base64Image = this.base64Image.replace(/%0D%0A/g, '\r\n');
      this.dialogController?.open();
      return true;
    }
    return false;
  })
```

**`methodList` is the security boundary, and it is a whitelist.** The injected
object is reachable from every script in the page under `window.picture`, but
only the named methods are callable, so a proxy object should carry exactly the
methods H5 needs and nothing else. Pairing that with `fileAccess(false)` and
`geolocationAccess(false)` means the page's only capability is the one method -
which is what makes shipping a local HTML file that runs unsandboxed JS
acceptable here.

**`onContextMenuShow` returning `true` is what replaces the menu**, not what
opens the sheet. Returning `false` (or falling off the end, as the document's
abridged snippet does) lets ArkWeb show its own copy/save menu on top of the
custom sheet. The `%0D%0A` replacement is the tell that the payload is MIME:
the Web layer percent-encodes the line breaks inside the data URL, and this puts
them back so the string matches what `encodeToStringSync(…, util.Type.MIME)`
produced.

**The H5 side — `entry/src/main/resources/rawfile/Photos.html`** (as shipped)

```html
<body onload="takePicture()">
<!-- fifteen <img class="gallery-image" id="myImageN"> without a src -->
<script>
    function takePicture() {
        let pictures = picture.chosePicture()
        for (let i = 0; i < pictures.length; i++) {
            let s = 'data:image/png;base64,' + pictures[i];
            document.getElementById('myImage' + i).src =  s;
        }
    }
</script>
```

The `<img>` tags ship with no `src` at all - the page is inert until the bridge
answers. `chosePicture()` is called synchronously and returns a real JS array,
which is worth noting: proxied methods that return a value are synchronous
across the bridge, so the fifteen files are read and encoded on the UI thread
during page load. Fifteen small PNGs is fine; a hundred photographs would not
be, and the fix there is `runJavaScript` pushing batches in rather than a
synchronous pull.

**The encoder — `entry/src/main/ets/pages/Photos.ets`** (as shipped)

```typescript
chosePicture() {
  let base64strings: string[] = [];
  let context = this.getUIContext().getHostContext() as common.UIAbilityContext;
  // 创建Base64编解码工具实例
  let base64helper = new util.Base64Helper();
  for (let i = 0; i < IMAGE_DATA.length; i++) {
    let arrayBuff = context.resourceManager.getRawFileContentSync(IMAGE_DATA[i]);
    // 将ArrayBuffer转换为带MIME类型的Base64字符串
    let base64string = base64helper.encodeToStringSync(arrayBuff, util.Type.MIME);
    base64strings.push(base64string);
  }
  return base64strings;
}
```

`util.Type.MIME` is the choice that propagates through the whole feature. MIME
output is RFC 2045: base64 broken into 76-character lines separated by CRLF.
That is why the page can embed it, why the context-menu handler has to
un-escape `%0D%0A`, and why the decoder must be told the same mode.

**The decoder — `entry/src/main/ets/utils/Base64Utils.ets`** (corrected, see `HW-15-0062`)

```typescript
static async base64ToArrayBuffer(base64Str: string): Promise<ArrayBuffer> {
  let base64 = new util.Base64Helper();
  let uint8Array: Uint8Array = await base64.decode(base64Str, util.Type.MIME);   // FIX: was decode(base64Str)
  let buffer: ArrayBuffer = uint8Array.buffer.slice(0, uint8Array.byteLength);
  return buffer;
}
```

**One missing argument breaks the app's only feature.** `decode` defaults to
`util.Type.BASIC`, whose alphabet has no place for the CRLFs that MIME mode
inserted - and that the context-menu handler carefully restored two files
earlier. Any image whose Base64 exceeds one 76-character line, which is every
real image, therefore fails to decode. The two modes must be chosen together;
the pairing is not a detail the caller can leave to defaults.

The `buffer.slice(0, byteLength)` is not superstition either: `Uint8Array.buffer`
may be a view into a larger allocation, and `fs.write` takes the whole
`ArrayBuffer`. Slicing to the exact byte length is what stops trailing garbage
from being appended to the PNG.

**The album write — `entry/src/main/ets/utils/PhotoUtils.ets`** (corrected, see `HW-15-0062`, `HW-15-0016`, `HW-15-0063`)

```typescript
// FIX: a plain class, not an unmounted @Component struct; UIContext passed in
export class PhotoUtils {
  async saveImageToAlbum(uiContext: UIContext, context: Context | undefined, imageBuffer: ArrayBuffer) {
    const prompt = uiContext.getPromptAction();
    if (context === undefined) {
      prompt.showToast({ message: $r('app.string.save_fail') });
      return;
    }
    let file: fs.File | null = null;
    try {
      let helper = photoAccessHelper.getPhotoAccessHelper(context);
      let uri = await helper.createAsset(photoAccessHelper.PhotoType.IMAGE, 'png');
      file = await fs.open(uri, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);   // FIX: was ||
      await fs.write(file.fd, imageBuffer);                                     // FIX: was un-awaited
      Logger.info('save image to system album success.');
      prompt.showToast({ message: $r('app.string.save_success') });             // FIX: was unconditional
    } catch (error) {
      Logger.error(`save image to system album failed, error.code:${error.code}, message: ${error.message}.`);
      prompt.showToast({ message: $r('app.string.save_fail') });
    } finally {
      if (file) {                                                               // FIX: was fs.close(file?.fd)
        fs.close(file.fd);
      }
    }
  }
}
```

**Four defects in eleven lines, and they compound.** `fs.write` returns a
promise; un-awaited, control reaches the `finally` and closes the fd while the
write is still in flight, so the asset in the gallery may be truncated or empty.
The inner `catch {}` is empty, so a rejection is not even logged. The success
toast sits outside the inner try and fires whatever happened. And the close is
`fs.close(file?.fd)` on a `file` that is still `null` if `fs.open` threw -
passing `undefined` to `close` raises an invalid-argument rejection that nobody
observes, which is the industry-wide pattern recorded as `HW-15-0016` (the same
shape appears in `UTIL-01`'s `PrintDocxPage` and in `PosterGen`). Awaiting the
write, guarding the close and moving the toast into the success path turns all
four into one honest outcome.

`fs.OpenMode.READ_WRITE || fs.OpenMode.CREATE` is a fifth, quieter bug: `||` on
two numbers yields the first truthy one, so the flag set is just `READ_WRITE`.
It happens not to matter because `createAsset` already created the file, but the
intent was `|`.

**The permission gate — `entry/src/main/ets/component/SavePopUp.ets`** (as shipped)

```typescript
const permissions: Array<Permissions> = ['ohos.permission.WRITE_IMAGEVIDEO'];

@CustomDialog
export struct SavePopUp {
  @Link base64Image: string;
  controller?: CustomDialogController;
  private context: common.UIAbilityContext = this.getUIContext().getHostContext() as common.UIAbilityContext;

  async saveImageToAlbum(base64Str: string) {
    if (await userPermissionUtils.reqPermissionsFromUser(this.context, permissions)) {
      let imageBuffer: ArrayBuffer = await Base64Utils.base64ToArrayBuffer(base64Str);
      photoUtils.saveImageToAlbum(this.getUIContext().getHostContext(), imageBuffer);
    }
  }
```

This is the shape to replace, not to copy (`HW-15-0063`).
`ohos.permission.WRITE_IMAGEVIDEO` is a **restricted** permission: declaring it
in `module.json5` is not enough, it needs an ACL entry in a signed provisioning
profile, and without one `requestPermissionsFromUser` resolves with a denial and
no dialog. On an ordinarily signed build the button silently does nothing. The
modern path needs no permission at all - `photoAccessHelper`'s
`showAssetsCreationDialog` lets the user confirm the save and hands back a
writable URI, and a `SaveButton` security component grants a one-shot write on
tap. Either removes the permission, the request, and the whole
`UserPermissionUtils` file.

Note too that `saveImageToAlbum` is fired and forgotten from the button handler,
which closes the dialog on the next line - so the sheet disappears before the
save has been attempted, and the outcome toast (once it is honest) arrives over
whatever is on screen by then.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.WRITE_IMAGEVIDEO",
    "reason": "$string:Write_ImageVideo_reason",
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": "inuse"
    }
  }
]
```

- `WRITE_IMAGEVIDEO` is `user_grant` **and restricted**. `reason` and
  `usedScene` are mandatory and present, but the grant additionally requires the
  permission to be listed in the app's ACL in a signed profile. See
  `documentation/harmonyos-guides/04_system/restricted-permissions.md`.
- No `ohos.permission.INTERNET` is declared, and none is needed: the Web loads
  `$rawfile` and the images never leave the app.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `javaScriptProxy` requires JavaScript to be enabled on the `Web` (the default)
  and the injected object is visible to **all** scripts in the page, including
  any third-party script the page might load. Keep `methodList` minimal and
  never inject anything holding credentials.
- Proxied method calls are synchronous from the H5 side, so anything expensive
  behind them blocks the page. Fifteen rawfiles is the ceiling this design
  comfortably carries.
- `onContextMenuShow` fires for links and editable fields too; the
  `ContextMenuMediaType.Image` check is what narrows it, and the `false` return
  in every other case is what keeps normal long-press behaviour working.
- `getSourceUrl().split(',')[1]` assumes a data URL. An `<img>` pointing at an
  `http(s)` or `resource://` URL would yield the wrong half or `undefined`;
  check the prefix before splitting if the page is not entirely under your
  control.
- The asset is always created as `'png'` regardless of the data URL's declared
  MIME type.

## Pitfalls

- **`HW-15-0062` (B/high, probable) — the save flow is doubly broken.**
  `Base64Utils` decodes in the default BASIC mode a payload that `Photos.ets`
  encoded with `util.Type.MIME` and whose CRLFs it deliberately restored, so any
  image longer than one 76-character line - effectively all of them - fails to
  decode; and in `PhotoUtils`, `fs.write` is not awaited, the fd is closed in a
  `finally` racing that write, the inner `catch` is empty and the success toast
  fires unconditionally. Fix: `decode(str, util.Type.MIME)`, `await fs.write`,
  and move the toast into the success branch.
- **`HW-15-0063` (B/medium, probable) — the feature depends on a restricted
  permission and reports failure through unmounted structs.** `createAsset`
  needs `WRITE_IMAGEVIDEO`, which is auto-denied without a signed ACL profile,
  so a normally signed build cannot succeed; and `PhotoUtils`,
  `UserPermissionUtils` and `PictureManagement` are `@Component` structs created
  with `new` and never mounted, so their `getUIContext()` calls - which sit
  exactly on the refusal and failure paths - have no backing UI instance. Fix:
  use `showAssetsCreationDialog` (or a `SaveButton`) instead of the permission,
  and pass `UIContext` / the host context in as parameters to plain classes.
- **`HW-15-0016` (B/low, confirmed) — systematic across three utilities samples:
  `closeSync`/`close` on a possibly-undefined file in a `finally`.** Here it is
  `fs.close(file?.fd)` after a failed `fs.open`, whose invalid-argument
  rejection is never observed and which masks the original error. Fix:
  `if (file) { fs.close(file.fd); }`.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web.md` - the `Web` component
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-basic-components-web
- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-attributes.md` - `javaScriptProxy`, `fileAccess`, `geolocationAccess`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-basic-components-web-attributes
- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-events.md` - `onContextMenuShow` and its return value
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-basic-components-web-events
- `documentation/harmonyos-references/02_application-framework/arkts-basic-components-web-webcontextmenuparam.md` - `getMediaType`, `getSourceUrl`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-basic-components-web-webcontextmenuparam
- `documentation/harmonyos-references/02_application-framework/js-apis-util.md` - `Base64Helper`, `encodeToStringSync`, `decode`, `util.Type`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-util
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `open`, `write`, `close`, `OpenMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-f.md` - `getPhotoAccessHelper`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-f
- `documentation/harmonyos-references/04_media/js-apis-photoaccesshelper.md` - `createAsset`, `showAssetsCreationDialog`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-photoaccesshelper
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` - the permission-free save path
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - why `WRITE_IMAGEVIDEO` needs an ACL
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `UTIL-01` - the same unguarded `finally` close (`HW-15-0016`)
