---
id: SOCIAL-38
title: Personalised QR code - a styled QRCode in a Stack, snapshotted whole and saved with SaveButton
industry: 14_social_communication
doc: huawei_industry_tree/14_social_communication/docs/38_customizable_qrcode.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/customizable_qrcode-0000002354787644
sample: huawei_industry_tree/14_social_communication/downloads/CustomizableQRCode.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [QRCode, Stack, SaveButton, SaveButtonOnClickResult, "UIContext.getComponentSnapshot", "componentSnapshot.get", "photoAccessHelper.getPhotoAccessHelper", createAsset, "fileIo.open", "fileIo.close", "image.createImagePacker", packToFile, bindSheet, Navigation, NavDestination, hilog]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-14-0078, HW-14-0087]
status: verified-with-fixes
---

## When to use

**Load this card when the user must be able to style something and then keep
the result as an image** - a personal QR code for adding friends, a shareable
profile card, a ticket, a coupon. The QR code is the excuse; the reusable
machinery is "compose a visual in ArkUI, snapshot the composed node, write the
PixelMap into the gallery".

The composition pattern is a `Stack` of three optional layers - background
image, the `QRCode` itself, a centre logo - wrapped in an outer `Column` that
also carries an optional title above or below. Each layer is toggled by an
`if` in the build tree and configured from a `bindSheet` panel that owns
nothing and writes back through `@Link`. That generalises to any live
"customise and preview" editor: the preview *is* the component tree, so there
is no separate render path and no risk of preview and export diverging.

The save path is where this sample needs care. `getComponentSnapshot().get(id)`
returns a `PixelMap` of a node addressed by `.id()`, and `SaveButton` grants a
one-shot save authorisation with **no permission in `module.json5`** - that
combination is worth copying verbatim. What is not worth copying is the
file-handle lifecycle around `packToFile`: **read `HW-14-0078` before shipping
this.**

## Feature checklist

- A first page takes the QR payload in a `TextArea` (max 512 characters) and
  pushes an edit page with the trimmed string as the navigation parameter.
- The edit page shows a live QR code that re-renders as the payload is typed
  in its own `TextInput`.
- A colour sheet offers a 前景色 / 背景色 (foreground / background) toggle and
  ten swatches; picking one repaints the QR code immediately.
- A title sheet takes up to 10 characters, a font colour and a size, and a
  switch that puts the title above or below the code.
- A logo sheet overlays one of four 30x30 icons at the centre of the code, or
  clears it.
- A background-image sheet puts a 200x200 image behind the code; when set, the
  QR background colour is forced transparent so the image shows through.
- The whole card - title included - has a fixed 231vp width and a height that
  grows from 231 to 250 when a title is present.
- A `SaveButton` in the title bar snapshots the card and writes it into the
  gallery as a PNG, then toasts 已保存到相册.

## Architecture

One `entry` module, six source files, two pages behind a `Navigation` /
`NavDestination` pair driven by a `routerMap`.

```
entry/src/main/ets
├── component/
│   ├── BackgroundImagePanelSheet.ets  4 images + "not set", writes @Link isBackgroundSet
│   ├── ColorPanelSheet.ets            fg/bg toggle + 10 swatches, writes two @Link colours
│   ├── LogoPanelSheet.ets             same shape as the background sheet, for the centre logo
│   └── TitlePanelSheet.ets            text, colour, size and the above/below switch
├── entryability/EntryAbility.ets      full screen, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
└── pages/
    ├── MainPage.ets                   @Entry: payload TextArea + Navigation host
    └── QRCodeEditPage.ets             NavDestination: the card, the four sheets, the save path
```

The documented 工程目录 lists `component`, `entryability` and `pages` but omits
`entrybackupability`, which is present in the zip and registered in
`module.json5`. Otherwise the tree matches.

**The design decision worth copying** is that every panel is a dumb `@Link`
writer and the edit page owns all thirteen state fields. `ColorPanelSheet` has
no state of its own at all:

```typescript
@Link isColorSetShow: boolean;
@Link isForegroundColor: boolean;
@Link qrCodeColor: string;
@Link qrCodeBackgroundColor: string;
```

so a swatch tap writes straight into `QRCodeEditPage.qrCodeColor` and the code
behind the half-open sheet repaints while the sheet is still up. That is the
whole reason to use `bindSheet` here rather than a full page: the preview stays
mounted and live under the sheet.

The two sheets that can be *cancelled* implement the reset in
`aboutToDisappear` rather than on a button - `LogoPanelSheet` clears
`logoResource` if `isLogoSet` was left false, and `TitlePanelSheet` restores
the default colour and size if the title was left empty. Cleanup on the sheet
closing, not on a decision the user never made.

## Implementation steps

1. **Declare the router map** (`"routerMap": '$profile:router_map'`) and export
   the edit page through a global `@Builder`
   (`qrCodeEditPageBuilder(name, param)`), so `MainPage` can `pushPath` by
   name with the payload as `param`.
2. **Read the payload in `onReady`**, from
   `context.pathInfo.param as string`, and keep `context.pathStack` for the
   back button.
3. **Cap and trim the payload.** `maxLength(512)` on both inputs and a
   `trim()` on submit - the reference is explicit that content beyond 512
   characters is truncated and that empty/`null` content produces an invalid
   code.
4. **Stack the three layers** background / `QRCode` / logo, each behind an
   `if`, with `alignContent: Alignment.Center`.
5. **Force the QR background transparent when a background image is set**,
   otherwise the opaque `backgroundColor` hides the image.
6. **Give the composed card two ids**: `'qrcode'` on the inner `Stack` and
   `'wholeQRCode'` on the outer `Column`. The snapshot is taken of the outer
   one so the title is included.
7. **Save with `SaveButton`**, and act only when
   `result === SaveButtonOnClickResult.SUCCESS`; the temporary authorisation
   this grants is why no gallery permission appears in `module.json5`.
8. **Create the asset, open it, pack into it, and close the fd only after the
   pack callback has fired** (`HW-14-0078`). `packToFile` is
   callback-asynchronous; a `finally` block closes the descriptor while the
   encoder is still writing.
9. **Await the save and catch its rejection** so a refused authorisation or a
   full disk reaches the user instead of an unhandled promise
   (`HW-14-0078`).

## Verified snippets

All snippets are from `CustomizableQRCode.zip`. Corrected forms are marked.

**The layered card — `entry/src/main/ets/pages/QRCodeEditPage.ets`** (as shipped)

```typescript
Column() {
  if (this.isTitleTop && this.title !== '') {
    Text(this.title)
      .align(Alignment.Center)
      .height(29)
      .fontSize(this.titleSize)
      .fontColor(this.titleColor)
      .opacity(0.9);
  }

  Stack({ alignContent: Alignment.Center }) {
    // BackgroundImage
    if (this.isBackgroundImageSet && this.backgroundImageResource !== undefined) {
      Image(this.backgroundImageResource)
        .width(200)
        .height(200);
    }

    // QRCode
    QRCode(this.qrCodeContent)
      .width(200)
      .height(200)
      .color(this.qrCodeColor)
      .backgroundColor(this.isBackgroundImageSet ? '#00ffffff' : this.qrCodeBackgroundColor); // 透明色

    // Logo
    if (this.isLogoSet && this.logoResource !== undefined) {
      Image(this.logoResource)
        .width(30)
        .height(30);
    }
  }
  .id('qrcode')
  .width(231)
  .height(this.title === '' ? 231 : 217)
  .borderRadius(12);

  if (!this.isTitleTop && this.title !== '') {
    Text(this.title)
      // ...
  }
}
.id('wholeQRCode')
.borderRadius(12)
.backgroundColor('rgb(255, 255, 255)')
.width(231)
.height((this.title !== '') ? 250 : 231);
```

**Three details carry the design.** The declaration order inside the `Stack` is
the z-order: background image first, code second, logo last, so the logo sits
on top of the modules at the centre - which is exactly where a QR code's error
correction can afford to lose data. The `'#00ffffff'` ternary is not cosmetic:
`QRCode` paints its own background, so without forcing alpha zero the
background image below it would be completely covered.

The two `.id()` calls are the export contract. `'qrcode'` names the code
alone; `'wholeQRCode'` names the card including the title, and it is the one
the snapshot uses. The height arithmetic (`231` alone, `217 + 29 + gaps = 250`
with a title) exists because both are fixed-size - a snapshot of an
auto-sized node would vary with the title length.

Both title branches render the same `Text` with the same attributes; only the
position in the `Column` differs. That is why `isTitleTop` is a boolean on the
page and not a property of the text.

**The save path — same file** (corrected, see `HW-14-0078`)

```typescript
async savePixelMapToAlbum(pixelMap: image.PixelMap) {
  let helper = photoAccessHelper.getPhotoAccessHelper(this.context);
  try {
    // FIX: createAsset was called outside the try - a refusal rejected unhandled
    let uri = await helper.createAsset(photoAccessHelper.PhotoType.IMAGE, 'png');
    let file: fs.File = await fs.open(uri, fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE);
    let imagePackerApi = image.createImagePacker();
    let packOpts: image.PackingOption = { format: 'image/png', quality: 98 };
    imagePackerApi.packToFile(pixelMap, file.fd, packOpts, (err: BusinessError) => {
      // FIX: close inside the callback, not in a finally that runs immediately
      fs.close(file.fd);
      if (err) {
        hilog.error(0x0000, 'QRCodeEditPage',
          `Failed to pack the image to file.code ${err.code},message is ${err.message}`);
        return;
      }
      hilog.info(0x0000, 'QRCodeEditPage', 'Succeeded in packing the image to file.');
      imagePackerApi.release((err: BusinessError) => {
        if (err) {
          hilog.error(0x0000, 'QRCodeEditPage',
            `Failed to release the image source instance.code ${err.code},message is ${err.message}`);
        }
      });
      this.promptAction.showToast({
        message: this.context.resourceManager.getStringByNameSync('saved_to_the_album_successfully')
      });
    });
  } catch (err) {
    hilog.error(0x0000, 'QRCodeEditPage',
      `An error occurred error.code ${err.code},message is ${err.message}`);
  }
}
```

**`packToFile` is callback-async, and that is the whole bug.** In the shipped
form the call returns immediately, the `try` block ends, and `finally` runs
`fs.close(file.fd)` while the encoder is still writing into that descriptor.
Whether the saved PNG is complete, truncated or empty depends on how fast the
encode finishes relative to the microtask that drains the `try` - and the
success toast inside the callback fires either way, so the user is told the
save worked. Closing inside the callback, on both the error and success
branches, is the minimal fix; the descriptor's lifetime then actually covers
the write.

The second half of the finding is the `await` boundary. In the shipped code
`createAsset` is awaited **outside** the `try`, so a user who dismisses the
save authorisation gets an unhandled rejection and no feedback at all. Moving
it inside is one line and turns a silent failure into a logged one -
production code should replace the `hilog.error` with a visible toast.

**Snapshot and save, permission-free — same file** (as shipped)

```typescript
private saveButtonOptions: SaveButtonOptions = {
  text: SaveDescription.SAVE
};

SaveButton(this.saveButtonOptions)
  .width(32)
  .height(21)
  .backgroundColor('rgb(241, 243, 245)')
  .onClick(async (event: ClickEvent, result: SaveButtonOnClickResult) => {
    const PIXEL_MAP = await this.getUIContext().getComponentSnapshot().get('wholeQRCode');
    if (result === SaveButtonOnClickResult.SUCCESS && PIXEL_MAP !== undefined) {
      this.savePixelMapToAlbum(PIXEL_MAP);   // FIX: should be `await`ed inside a try/catch
    }
  });
```

**`SaveButton` is a security component, and the `result` argument is the
authorisation.** The user's tap on the *system-rendered* button is what grants
a temporary write to the media library; that is why this sample writes into
the gallery with an empty `requestPermissions` array. Two rules follow, and
both are easy to break: the button's appearance may not be tampered with
beyond what the security-component API allows (an over-styled or overlapped
button is rejected at runtime), and the grant is valid only for this click -
you cannot cache `result` and save later.

`getComponentSnapshot().get('wholeQRCode')` resolves against the id set with
`.id()`, and it captures the node as rendered - background image, colours,
logo and title all included, which is precisely why the composition and the
preview are the same tree. Note that the call is awaited *before* the
`result` check; harmless here, but the snapshot is taken even when the user
declines.

**A panel that owns nothing — `entry/src/main/ets/component/LogoPanelSheet.ets`**
(as shipped)

```typescript
@Component
export struct LogoPanelSheet {
  @Link isLogoSet: boolean;
  @Link logoResource: Resource | undefined;

  aboutToDisappear(): void {
    if (!this.isLogoSet) {
      this.logoResource = undefined;
    }
  }

  build() {
    Column({ space: 16 }) {
      Row() {
        Image($r('app.media.not_set'))
          .width(59)
          .height(59)
          .onClick(() => {
            this.isLogoSet = false;
            this.logoResource = undefined;
          });

        Image($r('app.media.chose'))
          .width(59)
          .height(59)
          .onClick(() => {
            this.isLogoSet = true;
            this.logoResource = $r('app.media.chose');
          });
        // ... three more identical swatches
      }
      .width('100%')
      .height(59)
      .justifyContent(FlexAlign.SpaceBetween);
    }
    .backgroundColor('rgb(241, 243, 245)')
    .width('100%')
    .height('100%')
    .padding(16);
  }
}
```

Two `@Link`s and no `@State` is the entire component contract: the sheet
cannot get out of step with the preview because there is nothing to get out of
step. The paired flag and resource (`isLogoSet` + `logoResource`) rather than a
single nullable is what lets the build tree's `if` read as an intent rather
than a null check, and the `aboutToDisappear` guard covers the one case the
clicks do not - the sheet dismissed by swipe after a partial interaction.
`BackgroundImagePanelSheet` is the same component with different link names.

## Permissions & config

**None declared.** `module.json5` has no `requestPermissions` array at all, and
that is the point of the sample: writing a file into the user's gallery
normally needs `ohos.permission.WRITE_IMAGEVIDEO`, but `SaveButton` plus
`photoAccessHelper.createAsset` supplies a per-click temporary grant instead.

Other config that matters:

```json5
"routerMap": '$profile:router_map',
"pages": "$profile:main_pages",
"deviceTypes": ["phone", "tablet", "2in1"]
```

`EntryAbility` sets `COLOR_MODE_LIGHT`, calls `setWindowLayoutFullScreen(true)`,
reads both avoid areas into `AppStorage` (`topRectHeight`, `bottomRectHeight`)
and keeps them current with an `avoidAreaChange` listener. `MainPage` consumes
them through `@StorageProp` and `safeAreaPadding` with `px2vp`. The listener is
never unregistered in `onWindowStageDestroy` - the same boilerplate omission
that recurs across this industry's samples.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- QR payload: at most 512 characters; longer content is truncated. `null`,
  `undefined` and the empty string are invalid and produce an unusable code -
  and the edit page renders `QRCode('')` happily while the user clears the
  input.
- **Styling can make the code unscannable.** The document says so outright
  (若个性化二维码扫描后无法识别，请调整相关参数设置) and the sample enforces
  nothing: the ten foreground and ten background swatches can be combined into
  a pale-on-pale code, and a background image sits directly under the modules.
  Contrast and logo size are the developer's responsibility.
- The logo is a fixed 30x30 over a 200x200 code (2.25% of the area), which is
  within what the QR error correction tolerates; enlarging it is not safe.
- The four panel sheets offer fixed palettes and four bundled images. There is
  no colour picker and no gallery import.
- The saved file is always PNG (`createAsset(..., 'png')` and
  `format: 'image/png'`), at `quality: 98` - a parameter PNG ignores.

## Pitfalls

- **`HW-14-0078`** (B/high, confirmed): `savePixelMapToAlbum` closes the file
  descriptor in a `finally` block that runs as soon as `packToFile` is
  *called*, not when it completes, so the encoder writes into a descriptor
  that may already be closed - the saved QR image can be truncated or empty
  while the late callback still toasts success. The same method awaits
  `createAsset` outside the `try` and the caller does not await
  `savePixelMapToAlbum` at all, so a declined authorisation rejects unhandled
  with no feedback. Fix: move `fs.close(file.fd)` into the `packToFile`
  callback, widen the `try` to cover `createAsset`, and `await` the save in
  the `SaveButton` handler with a `catch`.

## References

- `huawei_industry_tree/14_social_communication/docs/38_customizable_qrcode.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customizable_qrcode-0000002354787644
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-qrcode.md` - `QRCode`, `color`, `backgroundColor`, the 512-character limit
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-qrcode
- `documentation/harmonyos-references/02_application-framework/ts-container-stack.md` - `Stack` and `alignContent`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-stack
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` - `SaveButtonOptions`, `SaveButtonOnClickResult`, the temporary grant
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `documentation/harmonyos-guides/03_application-framework/arkts-uicontext-component-snapshot.md` - `getComponentSnapshot().get(id)`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-uicontext-component-snapshot
- `documentation/harmonyos-references/04_media/js-apis-image.md` - `createImagePacker`, `packToFile`, `PackingOption`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-image
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `getPhotoAccessHelper`, `createAsset`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `open`, `close`, `OpenMode`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - `bindSheet` and `SheetOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
