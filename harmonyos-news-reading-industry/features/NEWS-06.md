---
id: NEWS-06
title: Save a Base64 image to the gallery - SaveButton grants the write, so no album permission is declared
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/06_base64_image_save.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/base64_image_save-0000002271203733
sample: huawei_industry_tree/11_news_reading/downloads/base64imageSave.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [fs, hilog, photoAccessHelper, util, window]
permissions: [ohos.permission.WRITE_IMAGEVIDEO]
min_api: 20
modules: [entry (entry)]
findings: [HW-11-0007, HW-11-0008]
status: verified-with-fixes
---

## When to use

Load this card when the app holds image bytes **it produced or received
in-process** - a Base64 blob from an API response, a rendered chart, a
screenshot of a card - and the user should be able to drop it into the system
gallery. The pattern: decode to an `ArrayBuffer`, then let a `SaveButton`
security control open a one-shot write window in which
`photoAccessHelper.createAsset` is allowed to run.

The point of `SaveButton` is that it replaces a permission. A user tapping a
system-rendered save control *is* the authorisation, so the app never declares
`ohos.permission.WRITE_IMAGEVIDEO` - a restricted permission that ordinary
apps cannot get approved anyway. This sample declares it regardless, which
contradicts its own opening sentence (`HW-11-0008`); delete it.

It generalises to every "save this to the user's library" affordance where the
app is not a gallery manager: save a coupon, save a QR code, save a generated
poster. The moment you need to *enumerate* or *modify* the user's existing
photos, `SaveButton` is no longer enough and you are back to the permission
route.

## Feature checklist

- An article page renders a Base64 image inline with no file ever touching
  disk.
- Tapping the image opens a bottom sheet with 取消 (cancel) and a system save
  control.
- The save control is a real `SaveButton`, drawn by the system with the
  `SaveDescription.SAVE_IMAGE` wording.
- Tapping it decodes the Base64 string, creates a new gallery asset, and
  writes the bytes into it.
- A toast confirms success; a failure raises an alert dialog instead.
- The sheet closes on tap-outside or back, and after a save.
- The app declares no album permission and the user is never shown a
  permission dialog.

## Architecture

One `entry` module, one page, one custom dialog, two static utility classes.
436 lines total, and most of one file is the Base64 literal.

```
entry/src/main/ets
├── common/Constants.ets        ImageConstants.BASE64_IMAGE - one ~40KB PNG as a string
├── entryability/EntryAbility.ets  full-screen layout, status-bar height -> AppStorage('topHeight')
├── log/Logger.ets
├── pages/MainPage.ets          @Entry, the article, the CustomDialogController
├── pages/SavePopUp.ets         @CustomDialog: cancel + SaveButton
└── utils/Base64Utils.ets       base64ToArrayBuffer
    utils/PhotoUtils.ets        saveImageToAlbum (createAsset + open + write + close)
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that `PhotoUtils.saveImageToAlbum`
takes a `UIContext`, not a `Context`:

```typescript
static async saveImageToAlbum(uiContext: UIContext, imageBuffer: ArrayBuffer)
```

It needs the ability context for `getPhotoAccessHelper`, and it needs the UI
instance for the toast and the alert dialog - so it takes the object that can
produce both (`uiContext.getHostContext()`, `uiContext.getPromptAction()`,
`uiContext.showAlertDialog()`) and never reaches for a global. That is the
form the global-API replacement rules push you towards, and it means the
utility works unchanged in a multi-window or multi-instance app where a bare
`promptAction.showToast` would land on the wrong window.

**The decision worth avoiding** is that the caller drops the promise:

```typescript
// SavePopUp.ets
async saveImageToAlbum(base64Str: string) {
  let imageBuffer: ArrayBuffer = await Base64Utils.base64ToArrayBuffer(base64Str);
  PhotoUtils.saveImageToAlbum(this.uiContext, imageBuffer);   // not awaited, not caught
}
```

`saveImageToAlbum` is `async` and its result is discarded. That is survivable
only while the function cannot reject - and `HW-11-0007` makes it reject on
the *success* path. The two defects compound: the double close throws, nothing
catches it, and the user sees a success toast followed by an unhandled
rejection in the log.

## Implementation steps

1. **Strip any data-URI prefix before decoding.** `Base64Helper.decode` wants
   the payload only; `data:image/png;base64,` in front of it makes the decode
   fail. The sample keeps the bare payload in `Constants.ets` and prepends the
   prefix only where it renders (`Image('data:image/jpg;base64,' + ...)`).
2. **Decode to `ArrayBuffer`, not `Uint8Array`,** and slice to
   `byteLength` - `uint8Array.buffer` may be larger than the view.
3. **Put the `SaveButton` inside the dialog, not the page.** The security
   control's grant is scoped to its own click; the work must start from that
   handler.
4. **Create the asset first, then open it.** `createAsset` returns a
   media-library URI that only becomes writable inside the granted window;
   `fs.open` it with `READ_WRITE | CREATE`.
5. **Close the file exactly once** (`HW-11-0007`). Either close in the `try`
   and clear the variable, or leave the close to `finally` alone.
6. **Await the save in the caller and catch it,** so a failure surfaces
   instead of becoming an unhandled rejection.
7. **Do not declare `ohos.permission.WRITE_IMAGEVIDEO`** (`HW-11-0008`). It is
   restricted, app review will reject it for an ordinary app, and the whole
   point of the control is that you do not need it.
8. **Report both outcomes in the UI**: a toast on success, an alert on
   failure. A silent failure here is indistinguishable from success because
   the gallery is a different app.

## Verified snippets

All snippets are from `base64imageSave.zip`. Corrected forms are marked.

**Decode — `entry/src/main/ets/utils/Base64Utils.ets`** (as shipped)

```typescript
import { util } from '@kit.ArkTS';

export class Base64Utils {
  /**
   * @note 如果传base64的图片字符串，需要将“data:image/png;base64,”前缀剪切掉
   */
  static async base64ToArrayBuffer(base64Str: string): Promise<ArrayBuffer> {
    let base64 = new util.Base64Helper();
    let uint8Array: Uint8Array = await base64.decode(base64Str);
    let buffer: ArrayBuffer = uint8Array.buffer.slice(0, uint8Array.byteLength);
    return buffer;
  }
}
```

**The `slice(0, uint8Array.byteLength)` is not decoration.** `decode` returns a
`Uint8Array` view; its backing `ArrayBuffer` can be longer than the view's
`byteLength`. Handing the raw `.buffer` to `fs.write` would append whatever
trailing bytes the allocator left behind, producing a file that is larger than
the image and, for some decoders, corrupt. Slicing to `byteLength` copies
exactly the decoded region.

The JSDoc note is the API's real contract, spelled out in Chinese: strip the
`data:image/png;base64,` prefix before calling. The sample honours it by
storing the payload bare and adding the prefix only at the `Image()` call
site, which is the right way round - one canonical form in state, presentation
prefixes added where needed.

**The save control — `entry/src/main/ets/pages/SavePopUp.ets`** (as shipped, with the caller's defect visible)

```typescript
@CustomDialog
export struct SavePopUp {
  @Link base64Image: string;
  uiContext = this.getUIContext();
  controller?: CustomDialogController;

  async saveImageToAlbum(base64Str: string) {
    let imageBuffer: ArrayBuffer = await Base64Utils.base64ToArrayBuffer(base64Str);
    PhotoUtils.saveImageToAlbum(this.uiContext, imageBuffer);
  }

  build() {
    Column() {
      // ... handle bar, title row with a close cross ...
      Row() {
        Button($r('app.string.Cancel'))
          .onClick(() => { this.controller?.close(); })
          .width(165).height(40)
          .backgroundColor('#F1F3F5')
          .fontColor('#0A59F7');

        SaveButton({
          text: SaveDescription.SAVE_IMAGE
        })
          .onClick(() => {
            this.saveImageToAlbum(this.base64Image);
            this.controller?.close();
          })
          .width(165).height(45);
      }
      .justifyContent(FlexAlign.SpaceBetween)
      .width('100%');
    }
  }
}
```

**`SaveButton` is a security component, and that constrains how you may style
it.** The system draws the label from `SaveDescription.SAVE_IMAGE`, not from
your string resource, and the platform refuses to honour styling that would
disguise the control - obscured, tiny, transparent or off-screen buttons stop
granting. Sizing it (165x45) and placing it beside an ordinary `Button` for
cancel, as here, is inside the allowed envelope. Note the asymmetry: the
cancel button is 40 tall and the save control 45, because the control has its
own internal padding.

**The grant is bound to this click.** `createAsset` will succeed only while
the click's authorisation window is open, which is why the whole save path
starts here rather than in `aboutToAppear` or behind a confirmation. Closing
the dialog on the same line is safe - the async work already holds what it
needs.

**Create, write, close — `entry/src/main/ets/utils/PhotoUtils.ets`** (corrected, see `HW-11-0007`)

```typescript
import { photoAccessHelper } from '@kit.MediaLibraryKit';
import fo from '@ohos.file.fs';
import { Logger } from '../log/Logger';

export class PhotoUtils {
  static async saveImageToAlbum(uiContext: UIContext, imageBuffer: ArrayBuffer) {
    let file: fo.File | undefined = undefined;
    try {
      let helper = photoAccessHelper.getPhotoAccessHelper(uiContext.getHostContext());
      let uri = await helper.createAsset(photoAccessHelper.PhotoType.IMAGE, 'png');
      file = await fo.open(uri, fo.OpenMode.READ_WRITE | fo.OpenMode.CREATE);
      await fo.write(file.fd, imageBuffer);
      Logger.info('save image to system album success.');   // FIX: the try-path close is removed
      uiContext.getPromptAction().showToast({
        message: $r('app.string.dialog1'),
        duration: 2000
      });
    } catch (error) {
      Logger.error(`save image to system album failed, error.code:${error.code}, error message: ${error.message}.`);
      uiContext.showAlertDialog({ message: $r('app.string.dialog2') });
    } finally {
      if (file) {
        fo.closeSync(file);          // now the only close
      }
    }
  }
}
```

**The shipped version closes twice.** It calls `await fo.close(file.fd)` at the
end of the `try` and then, because `file` is still a truthy `File`, the
`finally` runs `fo.closeSync(file)` on a descriptor that no longer exists.
That throws from inside `finally`, *after* the success toast has already been
shown, and since the caller neither awaits nor catches, it lands as an
unhandled rejection on every successful save. Two fixes are equivalent: drop
the inner close (shown above), or keep it and set `file = undefined`
immediately after. Dropping the inner one is simpler because `finally` already
covers the throw path.

**`createAsset(PhotoType.IMAGE, 'png')` is where the permission would
otherwise be needed.** The extension argument decides the file type the media
library registers, so it must match the actual bytes - this sample's payload
is a PNG (it begins `iVBORw0KGgo`), so `'png'` is right, even though
`MainPage.ets` renders it through a `data:image/jpg;base64,` prefix. That
prefix mismatch is cosmetic for the decoder but confusing to read; prefer
`data:image/png;base64,` for the same bytes.

**The declaration to delete — `entry/src/main/module.json5`** (corrected, see `HW-11-0008`)

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "pages": "$profile:main_pages",
    // FIX: the shipped file declares ohos.permission.WRITE_IMAGEVIDEO here.
    // SaveButton exists to avoid it. Remove the whole requestPermissions block.
    "abilities": [ /* EntryAbility ... */ ]
  }
}
```

The document's own opening line is 本示例在无需申请相册管理模块权限的情况下 -
"without applying for album-module permissions". The shipped `module.json5`
contradicts it directly with a `requestPermissions` entry for
`ohos.permission.WRITE_IMAGEVIDEO`, complete with `reason` and `usedScene`. A
reader copying the sample wholesale inherits a restricted ACL permission they
do not need and cannot justify at review.

## Permissions & config

**As shipped:** `ohos.permission.WRITE_IMAGEVIDEO`, `when: "inuse"`, reason
`$string:Write_ImageVideo_reason`.

**As it should ship:** none.

`WRITE_IMAGEVIDEO` is a restricted permission intended for gallery-class
apps; ordinary apps need an ACL grant to hold it at all. Everything this
sample does - create one new asset and write bytes into it - is covered by the
`SaveButton` grant. Keep the permission only if the app also needs to modify
or delete assets it did not create in the same interaction.

`deviceTypes` is `["phone", "tablet", "2in1"]`. `AppScope/app.json5` declares
no `configuration` profile, so the app follows the system font scale (contrast
with `NEWS-05`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **The image is a ~40KB Base64 literal compiled into `Constants.ets`.** That
  is fine for a demo and wrong for a product: the string is loaded into memory
  as part of the module and decoded on every save. Real payloads should arrive
  from the network or a file.
- `MainPage.ets` reads `AppStorage.get<number>('topHeight') as number` at
  field-initialiser time, and `EntryAbility` writes that key inside an `async
  onWindowStageCreate` after an `await`. The ordering happens to work because
  `loadContent` is called last, but the cast hides the `undefined` case - a
  `@StorageProp('topHeight') topHeight: number = 0` would be safer, as
  `NEWS-08` does for the same value.
- The save control's label is system-supplied. Localising it is not your
  decision; `SaveDescription.SAVE_IMAGE` picks the platform wording for the
  current locale.
- No progress indication. A large buffer writes silently and the toast is the
  first feedback; for multi-megabyte images add a spinner around the awaited
  save.

## Pitfalls

- **`HW-11-0007` (B/medium, confirmed) — the file descriptor is closed
  twice on the success path.** `PhotoUtils.ets:34` closes inside `try`, then
  `finally` closes the same `File` again because `file` is still set; the
  second close throws after the success toast, and the caller neither awaits
  nor catches, so every successful save produces an unhandled rejection. The
  document teaches the same shape. Fix: remove the close inside `try` and keep
  only the `finally` close (or set `file = undefined` after the inner close).
- **`HW-11-0008` (D/medium, confirmed) — the doc's headline claim is "no
  album permission needed", yet `module.json5:17` declares the restricted
  `ohos.permission.WRITE_IMAGEVIDEO`.** The declaration contradicts the whole
  reason `SaveButton` exists, misleads readers about what the control
  achieves, and adds a permission app review will reject for ordinary apps.
  Fix: delete the `requestPermissions` entry - the SaveButton-scoped
  `createAsset` flow needs none.

## References

- `documentation/harmonyos-guides/04_system/savebutton.md` - what the save control grants and the styling rules that keep it functional
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/savebutton
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` - `SaveButton`, `SaveDescription`, `SaveIconStyle`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `documentation/harmonyos-guides/05_media/photoaccesshelper-savebutton.md` - the createAsset-inside-SaveButton flow end to end
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-savebutton
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `getPhotoAccessHelper`, `createAsset`, `PhotoType`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-references/02_application-framework/js-apis-util.md` - `util.Base64Helper` and `decode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-util
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `open`, `write`, `close`, `closeSync`, `OpenMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `huawei_industry_tree/11_news_reading/docs/06_base64_image_save.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/base64_image_save-0000002271203733
