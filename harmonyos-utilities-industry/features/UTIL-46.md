---
id: UTIL-46
title: Poster generator - compose an image, a QR code and text on a drawing.Canvas and hand the result to SaveButton
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/46_poster_gen.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/poster_gen-0000002569022355
sample: huawei_industry_tree/15_utilities/downloads/PosterGen.zip
kits: ["@kit.AbilityKit", "@kit.ArkGraphics2D", "@kit.ArkTS", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit", "@kit.ScanKit"]
apis: [common2D, drawing, fileIo, generateBarcode, hilog, image, photoAccessHelper, router, scanCore, text, util, window]
min_api: 20
modules: [entry]
findings: [HW-15-0097, HW-15-0098, HW-15-0099, HW-15-0016, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when the app has to **produce an image file the user did not
draw** - a share card, a ticket, a certificate, a receipt, a promotional
poster - out of a template plus a few fields. The pattern is: build an empty
`PixelMap` at the output resolution, wrap it in a `drawing.Canvas`, draw the
background image, the QR code and the text into it, then pack the result to PNG.

The alternative most apps reach for first is a `componentSnapshot` of an ArkUI
subtree. That is easier, but it is bound to screen density and to whatever is
laid out on screen. Drawing into a `PixelMap` gives you an output whose size you
choose, independent of the display, renderable while the page is not visible -
which is what you want the moment posters have to be 1080 px wide on every
device, or generated in a batch.

The transferable pieces here are the two text paths (`TextBlob` for one line you
can measure and place by hand, `ParagraphBuilder` for wrapped multi-line text)
and the save handoff: the poster is passed to a preview page as base64 and
written to the album only from inside a `SaveButton` click, so the app never
asks for a media-write permission at all. **Read `HW-15-0098` before adopting
the template** - as shipped, a poster with no uploaded background renders white
text on a white canvas.

## Feature checklist

- A form page with a title field (20 chars max, live counter), a content field
  (100 chars, live counter) and an upload area.
- Tapping the upload area opens the system photo picker and shows the chosen
  image; a 删除 (delete) link clears it.
- 生成海报 (generate poster) composes background, QR code, title and content on
  one canvas.
- The background image is scaled to the poster width and the poster height
  follows the scaled image.
- The title is shrunk to fit one line down to a floor of 24 px, and wraps to a
  centred paragraph if it still does not fit.
- The content is laid out as a right-aligned wrapped paragraph below the title.
- A QR code encoding a fixed string is stamped in the bottom-right corner.
- The result is packed to PNG, encoded base64 and shown on a preview page.
- A `SaveButton` on the preview page writes the PNG into the user's album and
  toasts 已保存至相册 (saved to the album).

## Architecture

One `entry` module, two pages, one constants file.

```
entry/src/main/ets
├── common/CommonConstants.ets       PosterConstants: layout numbers + Chinese UI strings
├── entryability/EntryAbility.ets    full-screen layout; avoid areas -> AppStorage
├── entrybackupability/
└── pages
    ├── PosterGenIndex.ets           the form AND the whole drawing pipeline (510 lines)
    └── PosterPreview.ets            base64 -> Image, plus the SaveButton
```

The documented tree matches the zip (the doc misspells `entrybackupablility`).

**The design decision worth avoiding** is putting the entire pipeline in the
page struct. `PosterGenIndex` is the form, the picker, the QR generator, the
image scaler, the canvas composer and the packer, and it keeps its intermediate
state (`bgPixelMap`, `qrPixelMap`, `qrWidth`, `qrHeight`) as component fields
that `generatePoster` mutates and releases as it goes. All three findings on
this sample are consequences of that one choice: state that should be local to
a render is instead long-lived and gets destroyed (`HW-15-0097`) or carried over
(`HW-15-0099`).

The shape to copy instead is a `PosterRenderer` that takes
`{ title, content, background }` and returns a `PixelMap`, owning nothing
between calls. Every value the drawing code needs - the canvas height, the
fitted font size, the QR bitmap - is then a local, and generating two posters in
a row cannot interfere.

What *is* worth copying is the handoff. The poster never becomes a file inside
the app: `packToData` gives an `ArrayBuffer`, `Base64Helper` turns it into a
string, `router.pushUrl` carries it to the preview page, and only the user's
tap on `SaveButton` - a security component the system trusts - opens an album
asset and writes it. `module.json5` declares no `requestPermissions` at all.

## Implementation steps

1. **Pick the background with `PhotoViewPicker`**, `maxSelectNumber = 1`,
   `MIMEType = IMAGE_TYPE`. The picker needs no permission - it returns a URI
   the app is temporarily allowed to read.
2. **Open, read and decode into a `PixelMap`**, then release the `ImageSource`
   and guard the `close` in the `finally` against the cancel path
   (`HW-15-0016`).
3. **Generate the QR once and keep it**: `generateBarcode.createBarcode` is
   asynchronous, so if you release it after every render you must `await` the
   regeneration before drawing the next poster (`HW-15-0097`).
4. **Compute the output size per render** - scale factor from the source width
   to `IMG_WIDTH`, height from the scaled image - into locals, never into class
   statics (`HW-15-0099`).
5. **Create the target `PixelMap` with an `ArrayBuffer` of
   `width * height * 4`** bytes, `RGBA_8888`, `editable: true`, and construct
   the `drawing.Canvas` over it.
6. **Fill the canvas, then draw back to front**: background, QR, title,
   content. Choose text colours that contrast with the fill you just applied
   (`HW-15-0098`).
7. **Guard the QR draw on the QR**, not on the background image
   (`HW-15-0097`).
8. **Fit the title with `TextBlob.bounds()`** in a loop that shrinks the font,
   and fall back to a `ParagraphBuilder` when the floor is reached.
9. **Pack with `ImagePacker.packToData`** and pass the base64 to the preview
   page.
10. **Save only from inside `SaveButton.onClick`,** after checking
    `result === SaveButtonOnClickResult.SUCCESS`.

## Verified snippets

All snippets are from `PosterGen.zip`. Corrected forms are marked.

**Picking and decoding the background — `entry/src/main/ets/pages/PosterGenIndex.ets`** (corrected, see `HW-15-0016`)

```typescript
async getAssetsPhoto() {
  let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
  photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
  photoSelectOptions.maxSelectNumber = 1;
  let photoPicker = new photoAccessHelper.PhotoViewPicker();
  let photoSelectResult: photoAccessHelper.PhotoSelectResult = await photoPicker.select(photoSelectOptions);
  let filePath = photoSelectResult.photoUris[0];
  if (!filePath) {
    return;                                   // FIX: cancelling the picker yields an empty array
  }
  let file: fs.File | undefined = undefined;
  try {
    file = fs.openSync(filePath, fs.OpenMode.READ_ONLY);
    let fileSize = fs.statSync(file.fd).size;
    let buffer = new ArrayBuffer(fileSize);
    fs.readSync(file.fd, buffer);
    const imageSourceObj: image.ImageSource = await image.createImageSource(buffer);
    this.bgPixelMap = imageSourceObj.createPixelMapSync();
    imageSourceObj.release();
    this.showDeleteText = true;
  } finally {
    if (file) {
      fs.closeSync(file);                     // FIX: shipped code closes `undefined`
    }
  }
}
```

**The picker is the permission story.** `PhotoViewPicker.select` runs in a
system process; the URI it hands back carries a temporary read grant for that
one file, which is why this sample reads album images while declaring no
`ohos.permission.READ_IMAGEVIDEO`. Reading it as a buffer and decoding with
`createImageSource(buffer)` (rather than passing the URI to `Image`) is what
puts the pixels somewhere `drawing.Canvas` can consume.

`imageSourceObj.release()` after `createPixelMapSync()` is correct and, notably,
missing from the doc's version of this same function. The `finally` is not: if
the user cancels, `photoUris` is empty, `openSync` throws on `undefined`, and
the `finally` then calls `closeSync(undefined)` - which throws again, replacing
the original error with an unhandled rejection in a caller that has no `catch`.
The same shape appears in two other utilities samples (`HW-15-0016`).

**The canvas composition — same file, `generatePoster`** (corrected, see `HW-15-0097`, `HW-15-0098`, `HW-15-0099`)

```typescript
async generatePoster() {
  if (!this.qrPixelMap) {
    await this.generateQRCode();              // FIX: shipped call is not awaited
  }
  // FIX: per-render locals - the shipped code mutates PosterConstants.IMG_HEIGHT
  // in resizeImage() and PosterConstants.TEXT_SIZE in the fit loop below.
  // resizeImage() here RETURNS the scaled height instead of writing the static.
  let imgHeight: number = this.resizeImage();
  let textSize: number = PosterConstants.TEXT_SIZE;

  const poster: image.PixelMap = this.createPixelMap(imgHeight, PosterConstants.IMG_WIDTH);
  const canvas: drawing.Canvas = new drawing.Canvas(poster);
  let color: common2D.Color = { alpha: 255, red: 255, green: 255, blue: 255 };
  canvas.drawColor(color);
  canvas.save();

  if (this.bgPixelMap) {
    canvas.drawImage(this.bgPixelMap, 0, 0);  // 添加选择的背景图
    canvas.save();
  }
  if (this.qrPixelMap) {                      // FIX: shipped guard is `if (this.bgPixelMap)`
    this.qrPixelMap.scaleSync(PosterConstants.QR_WIDTH / this.qrWidth,
      PosterConstants.QR_WIDTH / this.qrHeight);
    canvas.drawImage(this.qrPixelMap,
      PosterConstants.IMG_WIDTH - PosterConstants.QR_LEFT_PADDING - PosterConstants.QR_WIDTH,
      imgHeight - PosterConstants.QR_WIDTH - PosterConstants.QR_LEFT_PADDING);
    canvas.save();
    // FIX: shipped code release()s and nulls the QR here, so the next poster has none
  }

  // 绘制标题文字 - FIX: white on a white fill; use a colour that survives no background
  const titleColor: common2D.Color = this.bgPixelMap
    ? { alpha: 255, red: 255, green: 255, blue: 255 }
    : { alpha: 255, red: 0, green: 0, blue: 0 };
  const brush = new drawing.Brush();
  brush.setColor(titleColor);
  canvas.attachBrush(brush);
  const font = new drawing.Font();
  font.setSize(textSize);
  font.setEdging(drawing.FontEdging.ANTI_ALIAS);
  let textBlob = drawing.TextBlob.makeFromString(this.title, font, drawing.TextEncoding.TEXT_ENCODING_UTF8);
  let bounds = textBlob.bounds();
  let titleWidth: number = bounds.right - bounds.left;
  let titleHeight: number = bounds.bottom - bounds.top;
  // 通过循环找到标题合适字号，使得标题文字能在一行内显示完；同时设置限制字号为不低于24px
  while (titleWidth > PosterConstants.IMG_WIDTH - PosterConstants.TEXT_LEFT_PADDING * 2) {
    textSize--;
    if (textSize < 24) {
      break;
    }
    font.setSize(textSize);
    textBlob = drawing.TextBlob.makeFromString(this.title, font, drawing.TextEncoding.TEXT_ENCODING_UTF8);
    bounds = textBlob.bounds();
    titleWidth = bounds.right - bounds.left;
    titleHeight = bounds.bottom - bounds.top;
  }
  // ... single-line drawTextBlob, or a ParagraphBuilder when it still does not fit
  canvas.detachBrush();
  canvas.save();
}
```

**A brush is state on the canvas, not on the draw call.** `attachBrush(brush)`
installs the fill; everything drawn until `detachBrush()` uses it. That is why
the sample creates a second `Brush` (`brush1`) for the content text instead of
mutating the first: attach/detach pairs bracket each block of text, and the
colour a block is drawn in is decided before you enter the block, not inside it.
Both brushes here are pure white, and the canvas has just been filled pure
white, so the entire poster text is invisible unless a background image covers
it (`HW-15-0098`).

**The QR is the fragile part.** `createBarcode` returns a promise; the shipped
code calls `generateQRCode()` without awaiting it, releases the resulting
`PixelMap` at the end of every render, and guards the whole QR block on
`this.bgPixelMap` - a copy-paste of the background guard just above it. The
combination means the *first* poster with a background gets a QR and no other
poster ever does: the second render finds `qrPixelMap` undefined, fires an
un-awaited regeneration, and draws before it resolves. Even if it did resolve,
`qrWidth` is still `0` at that point, so the scale factor is `Infinity`. Keep
one QR bitmap for the life of the page, or await the regeneration.

**Fitting the title — same file** (as shipped, in the fit loop above)

The measure-shrink-remeasure loop is the right idea: `TextBlob.bounds()` gives
the ink box of the string at the current font size, so shrinking the font and
rebuilding the blob converges on a size that fits `IMG_WIDTH` minus twice the
padding. What makes it a trap is the accumulator: `PosterConstants.TEXT_SIZE` is
a **mutable class static** (`static TEXT_SIZE: number = 36`, not `readonly`) and
the loop decrements it in place with no reset. Generate a poster with a long
title and the constant is permanently 24 for the rest of the process - the next
short title renders tiny. `PosterConstants.IMG_HEIGHT` has the same defect via
`resizeImage()`, so poster two inherits poster one's canvas height
(`HW-15-0099`). Anything a render writes to must be a local.

When the floor is hit, the code switches from `TextBlob` to
`text.ParagraphBuilder` with `align: CENTER`, `maxLines: 30`,
`wordBreak: BREAK_WORD` and a `locale`, calls `layoutSync(TEXT_WIDTH)` to wrap,
reads `getHeight()` and uses it to push the content paragraph down. That
`titleHeight` feeding the content's y is the only vertical relationship in the
layout - everything else is a constant.

**Packing and saving — `entry/src/main/ets/pages/PosterPreview.ets`** (as shipped)

```typescript
// PosterGenIndex.generatePoster, end:
const imagePackerApi: image.ImagePacker = image.createImagePacker();
let packOpts: image.PackingOption = { format: 'image/png', quality: 100 };
imagePackerApi.packToData(poster, packOpts).then((data: ArrayBuffer) => {
  let base64 = new util.Base64Helper().encodeToStringSync(new Uint8Array(data));
  this.routePage(base64);                    // router.pushUrl to pages/PosterPreview
});

// PosterPreview:
async savePixelMapToAlbum() {
  let helper = photoAccessHelper.getPhotoAccessHelper(this.context);
  let uri = await helper.createAsset(photoAccessHelper.PhotoType.IMAGE, 'png');
  let file = await fileIo.open(uri, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
  let result = new util.Base64Helper().decodeSync(this.base64URI);
  try {
    await fileIo.write(file.fd, result.buffer);
    this.getUIContext().getPromptAction().showToast({ message: '已保存至相册！' });
  } finally {
    await fileIo.close(file.fd);
  }
}

SaveButton(this.saveButtonOptions)
  .onClick((event, result: SaveButtonOnClickResult) => {
    if (result === SaveButtonOnClickResult.SUCCESS) {
      this.savePixelMapToAlbum();
    }
  })
```

**`SaveButton` is why this app needs no permission.** It is a security
component: the system renders the button, and a click on it grants the app a
one-shot right to create an album asset. `createAsset` succeeds only inside
that grant, which is why the `SUCCESS` check is not optional - it is the
difference between the user's tap and a programmatic call. Compare `UTIL-27`,
which does the same save with a declared `WRITE_IMAGEVIDEO` and is auto-denied
without an ACL (`HW-15-0063`); the security component is the sanctioned path.

Carrying the poster as base64 through `router.pushUrl` is convenient at 328 px
wide and is the one thing here that does not scale: router params are not sized
for megabytes of image. For a real poster size, pass the sandbox path (or keep
the `PixelMap` in `AppStorage`) instead of the encoded bytes.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`; both media
interactions go through system-mediated components (`PhotoViewPicker`,
`SaveButton`). `deviceTypes` is `["phone", "tablet", "2in1", "wearable"]`.

`EntryAbility` sets `setWindowLayoutFullScreen(true)` and publishes
`avoidAreaTop` / `avoidAreaBottom` into `AppStorage`, which both pages read once
into plain fields and apply as padding. Two things follow. The values are read
once inside the `setWindowLayoutFullScreen` promise, so a page constructed
before that resolves gets `undefined` cast to `number`; and because the fields
are not `@StorageProp`, a fold, a rotation or a 2in1 window resize never updates
the padding - there is no `avoidAreaChange` subscription anywhere in the sample.
Compare `TOUR-03`, which reads the same two keys with `@StorageProp` and a typed
default of `0`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **Real devices only** - the doc states the sample does not run on the
  emulator (`ScanKit` barcode generation is unavailable there).
- The poster is a fixed 328 px wide (`IMG_WIDTH`), i.e. sized in the same
  numbers as the on-screen form. The output is a 328-px-wide PNG regardless of
  device density, so it is a preview-quality asset, not a shareable poster.
- Height comes from the scaled background; with no background it stays at the
  default `386`.
- The QR content is the hardcoded string `'This is a PosterGenerator.'`; there
  is no field for it.
- Title is capped at 20 characters by `maxLength`; the content counter says
  `/100` but the `TextArea` has no `maxLength`, so the limit is display-only.
- `TEXT_SIZE` floors at 24 px; a title long enough to still overflow at 24 px
  wraps to at most 30 lines.

## Pitfalls

- **`HW-15-0097`** (B/medium, confirmed): the second poster loses its QR code.
  The QR `PixelMap` is released and nulled at the end of each render, the
  regeneration on the next click is not awaited, and the QR block is guarded by
  `if (this.bgPixelMap)` - a copy of the background guard - so a poster without
  an uploaded image never gets a QR at all. `qrWidth` is `0` on the un-awaited
  path, making the scale factor `Infinity`. Doc 46 reproduces the same
  release/regenerate pattern. Fix: keep the QR for the life of the page (or
  await it) and guard on `qrPixelMap`.
- **`HW-15-0098`** (B/medium, confirmed): a poster with no background is white
  on white. The canvas is filled `255,255,255` and both the title brush and the
  content paragraph style are also pure white. Fix: dark default brushes, or a
  default background.
- **`HW-15-0099`** (B/medium, confirmed): `PosterConstants.TEXT_SIZE` and
  `PosterConstants.IMG_HEIGHT` are mutable statics used as constants. The
  title-fit loop decrements `TEXT_SIZE` and `resizeImage()` overwrites
  `IMG_HEIGHT`, and neither is ever restored - so a long-titled poster
  permanently shrinks the font for every later poster, and the canvas height
  carries the previous image's dimensions. Fix: copy them into locals per
  render.
- **`HW-15-0016`** (B/low, confirmed - systematic across three utilities
  samples): `fs.closeSync(file)` in a `finally` on a `file` that is still
  `undefined` when the picker was cancelled or the open threw. The cleanup
  error replaces the real one and surfaces as an unhandled rejection, because
  the `onClick` calling `getAssetsPhoto()` has no `catch`. Fix: guard the close,
  and return early on an empty `photoUris`.

## References

- `documentation/harmonyos-references/05_graphics/arkts-apis-graphics-drawing-canvas.md` -
  `Canvas`, `drawColor`, `drawImage`, `drawTextBlob`, `attachBrush`/`detachBrush`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-graphics-drawing-canvas
- `documentation/harmonyos-references/05_graphics/arkts-apis-graphics-drawing-textblob.md` -
  `TextBlob.makeFromString` and `bounds()`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-graphics-drawing-textblob
- `documentation/harmonyos-references/05_graphics/arkts-apis-graphics-drawing-font.md` - `setSize`, `setEdging`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-graphics-drawing-font
- `documentation/harmonyos-references/05_graphics/arkts-apis-graphics-drawing-brush.md` - `Brush.setColor`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-graphics-drawing-brush
- `documentation/harmonyos-references/05_graphics/js-apis-graphics-text.md` -
  `ParagraphBuilder`, `ParagraphStyle`, `TextStyle`, `layoutSync`, `paint`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-graphics-text
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` -
  `PhotoSelectOptions` and `select`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagepacker.md` - `packToData` and `PackingOption`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-references/04_media/arkts-apis-image-pixelmap.md` - `createPixelMapSync`, `scaleSync`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-pixelmap
- `documentation/harmonyos-guides/05_media/scan-barcodegenerate.md` and
  `documentation/harmonyos-references/04_media/scan-generatebarcode.md` - `createBarcode` and `CreateOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/scan-barcodegenerate
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/scan-generatebarcode
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` -
  `SaveButton`, `SaveButtonOptions`, `SaveButtonOnClickResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `documentation/harmonyos-guides/05_media/photoaccesshelper-savebutton.md` - the save-without-permission flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-savebutton
- `UTIL-08` - the web-page variant of this feature: render a poster from a `Web`
  component and share it
- `UTIL-27` - the same album save done with a declared permission, and why that fails
