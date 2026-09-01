---
id: PHOTO-14
title: Image format conversion - decode to an ImageSource, re-encode with packToFile, hand off to the document picker
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/14_image_converter.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_converter-0000002368877552
sample: huawei_industry_tree/18_photography/downloads/ImageConverter.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [common, fs, hilog, image, photoAccessHelper, picker, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0005, HW-18-0024, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when the app has to **turn one image file into another format** -
PNG to WebP, HEIC to JPEG, anything to anything the platform encoder supports -
and let the user save the result where they choose. It is the core of a
converter tool, and the same three-step shape (decode into an `ImageSource`,
re-encode with an `ImagePacker`, hand the file to a picker) covers export flows
in editors, compressors and share sheets.

The load-bearing idea is that **no pixel map is ever created**.
`ImagePacker.packToFile` accepts an `ImageSource` directly, so the image is
transcoded without a decode-to-bitmap round trip in ArkTS memory. That matters
for large photos: a 48 MP HEIC becomes a `PixelMap` of roughly 190 MB if you go
through `createPixelMap`, and zero bytes of it if you do not.

The other transferable piece is the **permission-free file path**. Both ends of
this flow are pickers - `DocumentViewPicker.select` to read and
`DocumentViewPicker.save` to write - so the app declares no permissions at all
and still reads and writes arbitrary user files. Prefer this to
`READ_MEDIA`/`WRITE_MEDIA` whenever the user is choosing the specific file.

## Feature checklist

- The home page shows a banner that opens the system document picker filtered to
  `.png`, `.jpeg`, `.jpg`, `.bmp`, `.webp`, `.dng`, `.heic`.
- Choosing a file navigates to a conversion page showing the picture full-bleed
  with a bottom sheet of format chips.
- The chip list is the supported output formats **minus the source format**, so
  the no-op conversion is not offered.
- The first remaining format is preselected; tapping a chip changes the
  selection and its border and fill.
- "开始转换" converts the file into the app sandbox, then opens the system save
  dialog with a suggested file name of `<basename>.<format>`.
- Confirming the dialog copies the converted file to the chosen location and
  shows a "成功保存至本地" toast.
- Cancelling the save dialog leaves no toast and no error.
- The four other home tabs are inert placeholders.

## Architecture

One `entry` module: two pages inside a `Navigation`, one singleton utility, one
model file.

```
entry/src/main/ets
├── entryability/EntryAbility.ets       full screen, avoid areas -> AppStorage
├── entrybackupability/
├── model/ImageParams.ets               IImageInfo {imageUrl, imageName} + OUTPUT_FORMATS
├── pages
│   ├── MainPage.ets                    @Entry, Navigation host, the picker entry point
│   └── FormatConversionPage.ets        NavDestination: chips, convert, save
└── utils/ImageUtil.ets                 singleton: pick, copy, decode, pack, save (248 lines)
```

**The documented tree does not match the zip.** The document lists
`pages/MinePage.ets` as 首页; the zip ships `pages/MainPage.ets`. It also names
the backup ability `EntryBackAbility.ets` where the zip has
`EntryBackupAbility.ets`. Both are naming drift, not missing files.

**The design decision worth copying** is the sandbox copy in `onReady`. The
picker returns a `file://docs/...` URI the app may only read; before doing
anything else the page copies it into `filesDir` under its original name:

```typescript
this.imageSandboxPath = this.uiAbilityContext.filesDir + '/' + this.imageName;
ImageUtil.getInstance().copyFile(this.imageUrl, this.imageSandboxPath);
```

Everything downstream - `fs.statSync`, `fs.readSync`, `packToFile` into a
sibling path - then works on ordinary sandbox paths with no URI permission
lifetime to worry about. The cost is one full copy of the source file; the
benefit is that the conversion cannot fail halfway because a temporary URI grant
expired.

`ImageUtil` being a singleton (`private constructor` + `getInstance()`) is
cosmetic here - it holds no state - but it does keep the format tables in one
place, which is where the real complexity lives.

## Implementation steps

1. **Pick the source with `DocumentViewPicker.select`,** passing
   `fileSuffixFilters` so the user cannot choose a file the decoder rejects, and
   `maxSelectNumber: 1`.
2. **Guard the empty result before indexing** (`HW-18-0024`). A cancelled picker
   resolves with `[]`, and `selectResult[0]` is then `undefined`.
3. **Derive the base name and extension by `lastIndexOf('.')`,** and
   `decodeURIComponent` the picker's URI segment - names with spaces or Chinese
   characters arrive percent-encoded.
4. **Copy the picked file into `filesDir`** and work from the sandbox path.
5. **Offer only the other formats:** `OUTPUT_FORMATS.filter(f => f !== sourceFormat)`.
6. **Read the file into an `ArrayBuffer` and `image.createImageSource(buffer)`.**
   Close the file descriptor in a `finally` - the sample does this correctly.
7. **Map the chip label to a MIME type and a quality,** and pass the result as
   `image.PackingOption` to `packToFile(imageSource, file.fd, packOpts)`.
8. **Release the `ImageSource` when the conversion finishes** (`HW-18-0005`),
   and release the `ImagePacker` in a `finally` - the sample already does the
   latter.
9. **Save with `DocumentViewPicker.save`,** seeding `newFileNames` with the
   converted name, and copy the sandbox file into the returned URI.
10. **Treat an empty save result as a cancel,** not an error - the sample gets
    this one right.

## Verified snippets

All snippets are from `ImageConverter.zip`. Corrected forms are marked.

**Decoding into an ImageSource — `entry/src/main/ets/utils/ImageUtil.ets`** (as shipped)

```typescript
public imageToImageSource(imagePath: string): image.ImageSource {
  let imageFile: fs.File | null = null;
  try {
    imageFile = fs.openSync(imagePath, fs.OpenMode.READ_ONLY);
    const STAT = fs.statSync(imagePath);
    const BUFFER = new ArrayBuffer(STAT.size);
    fs.readSync(imageFile.fd, BUFFER);

    const IMAGE_SOURCE = image.createImageSource(BUFFER);
    return IMAGE_SOURCE;
  } catch (error) {
    throw new Error('Failed to trans image.');
  } finally {
    if (imageFile) {
      try {
        fs.closeSync(imageFile);
      } catch (closeError) {
        hilog.error(0x0000, 'ImageUtil', `close file error: ${closeError.message}`);
      }
    }
  }
}
```

**The buffer overload is the deliberate choice here.** `createImageSource` also
takes a raw `fd`, which would avoid the `ArrayBuffer` entirely - but an
fd-backed source stays tied to an open descriptor for its whole life, and this
function wants to hand the caller a self-contained object and close the file
before returning. `fs.statSync(...).size` sizes the buffer exactly; there is no
chunked read, so the whole file is resident during the conversion.

The `finally` is a good model for the rest of this industry's samples:
`HW-18-0022` records five photography samples that open with `fs.openSync` and
never close. This one closes, and closes defensively - a throwing `closeSync`
inside `finally` would otherwise replace the real error.

**Re-encoding — same file** (as shipped)

```typescript
public async imageFormatTrans(imageSource: image.ImageSource, targetPath: string, format: string): Promise<void> {
  let myFormat: string = '';
  let myQuality: number = 98;
  switch (format) {
    case 'jpeg': { myFormat = 'image/jpeg'; break; }
    case 'webp': { myFormat = 'image/webp'; break; }
    case 'png':  { myFormat = 'image/png';  break; }
    case 'heic': { myFormat = 'image/heic'; break; }
    case 'jpg':  { myFormat = 'image/jpeg'; myQuality = 92; break; }
    default: { return; }
  }

  let packOpts: image.PackingOption = { format: myFormat, quality: myQuality };
  let file: fs.File | null = null;
  const IMAGE_PACKER_API: image.ImagePacker = image.createImagePacker();

  try {
    file = fs.openSync(targetPath, fs.OpenMode.CREATE | fs.OpenMode.READ_WRITE);
    try {
      await IMAGE_PACKER_API.packToFile(imageSource, file.fd, packOpts);
    } catch (err) {
      hilog.error(0x0000, 'ImageUtil',
        `Failed to pack the image to file (inside packToFile). Code: ${err.code}, Message: ${err.message}`);
    }
  } catch (err) {
    hilog.error(0x0000, 'ImageUtil', `Failed to open file for writing. Code: ${err.code}, Message: ${err.message}`);
  } finally {
    if (file) {
      fs.closeSync(file);
    }
    IMAGE_PACKER_API.release((err: BusinessError) => { /* logged */ });
  }
}
```

**Three details carry this function.** The `switch` is not decoration: the chip
label is a *file extension* and `PackingOption.format` wants a *MIME type*, and
`jpg` and `jpeg` map to the same encoder while carrying different quality
defaults (92 against 98). Encoding a `png` at `quality: 98` is harmless -
`quality` is ignored for lossless formats - so one table can serve both.

The nested try/catch is the right shape even though it reads oddly: the outer
one covers `fs.openSync` (a bad path, a full disk), the inner one covers
`packToFile` (an unsupported encoder on this device - HEIC in particular is not
universally available). Collapsing them would make a missing HEIC encoder
indistinguishable from a permission error in the log.

The `finally` closes the file **and** releases the packer. That release is the
exception in this industry: `HW-18-0036` records eight photography samples that
call `image.createImagePacker()` per save and never release it. Copy this
version, not theirs.

**Releasing the source — `entry/src/main/ets/pages/FormatConversionPage.ets`**
(corrected, see `HW-18-0005`)

```typescript
Button($r('app.string.start_trans'))
  .width('100%')
  .height(40)
  .onClick(async () => {
    let imageSource: image.ImageSource = ImageUtil.getInstance().imageToImageSource(this.imageSandboxPath);
    try {
      this.targetPath = this.uiAbilityContext.filesDir + '/' + this.imageBaseName + '.' + this.clickedFormat;
      await ImageUtil.getInstance().imageFormatTrans(imageSource, this.targetPath, this.clickedFormat);

      let newFileName: string = this.imageBaseName + '.' + this.clickedFormat;
      ImageUtil.getInstance().saveToLocalFile(this.uiAbilityContext, this.uiContext, newFileName, this.targetPath);
    } finally {
      imageSource.release();          // FIX: absent in the sample - native decoder memory leaks per conversion
    }
  });
```

**`ImageSource` holds native decoding resources that the ArkTS GC does not
account for.** The sample creates one on every tap of the convert button and
never releases it, so converting a folder of photos one by one grows the native
heap monotonically for the process lifetime. The official image-decoding guide
releases the source as the last step of every example.

`release()` belongs to the caller, not to `imageToImageSource`, because the
source has to outlive that function - `packToFile` consumes it. Wrapping the
whole conversion in `try/finally` also covers the case where `imageFormatTrans`
rejects: the source is released either way. The same defect is recorded across
eight photography files, including `PictureSticker`, `SubjectSegmentation`,
`GIFGenerator` and `Picture` (`HW-18-0005`).

**Picking the source, cancel-safe — `entry/src/main/ets/utils/ImageUtil.ets`**
(corrected, see `HW-18-0024`)

```typescript
public async selectImageFromDocument(context: common.UIAbilityContext): Promise<IImageInfo> {
  try {
    let documentPicker = new picker.DocumentViewPicker(context);
    let supportedFormat: string[] = ['.png', '.jpeg', '.jpg', '.bmp', '.webp', '.dng', '.heic'];
    let selectResult: string[] =
      await documentPicker.select({ fileSuffixFilters: supportedFormat, maxSelectNumber: 1 });

    if (!selectResult || selectResult.length === 0) {     // FIX: cancel resolves with an empty array
      return { imageUrl: '', imageName: '' };
    }

    let url: string = selectResult[0];
    let index = url.lastIndexOf('/') + 1;
    let fileName: string = decodeURIComponent(url.slice(index));

    let imageInfo: IImageInfo = { imageUrl: url, imageName: fileName };
    return imageInfo;
  } catch (err) {
    hilog.error(0x0000, 'ImageUtil', `PhotoViewPicker failed with err: ${err.code}, ${err.message}`);
    throw new Error('Failed to select image.');
  }
}
```

**Cancelling a picker is a routine action, not an error.** Without the guard,
`url` is `undefined`, `url.lastIndexOf` throws a `TypeError`, the `catch`
converts it into `Error('Failed to select image.')`, and that rejection travels
into `MainPage`'s `async onClick` - which has no `.catch`, so every cancel
produces an unhandled promise rejection. Returning an empty `IImageInfo` fits
the caller, which already tests `if (imageInfo.imageUrl !== '')` before pushing
the destination page.

The `catch` here is also worth a note as an anti-pattern: it swallows the
`BusinessError` and rethrows a bare `Error`, so the caller cannot distinguish a
cancel from a real picker failure. Keep the original error, or type the return.

`decodeURIComponent` on the last path segment is what makes non-ASCII file names
work; the picker returns percent-encoded URIs, and the decoded name is reused
verbatim as the sandbox file name and as the save dialog's suggestion.

**Saving through the document picker — same file** (as shipped)

```typescript
public saveToLocalFile(uiAbilityContext: common.UIAbilityContext, uiContext: UIContext, newFileName: string,
  sourcePath: string) {
  if (newFileName === '') {
    return;
  }
  let documentPicker = new picker.DocumentViewPicker(uiAbilityContext);
  let documentSaveOptions = new picker.DocumentSaveOptions();
  documentSaveOptions.newFileNames = [newFileName];
  documentPicker.save(documentSaveOptions).then((documentSaveResult: string[]) => {
    if (!documentSaveResult || documentSaveResult.length === 0 || !documentSaveResult[0]) {
      hilog.info(0x0000, 'saveToFile', 'User canceled the save operation.');
      return;                                    // cancel: quiet, no toast
    }
    documentSaveResult.forEach((path: string) => {
      this.copyFile(sourcePath, path);
    });
    uiContext.getPromptAction().showToast({ message: `成功保存至本地`, duration: 1000 });
  }).catch((err: BusinessError) => {
    hilog.error(0x0000, 'saveToFile', 'DocumentViewPicker.save failed with err: ' + JSON.stringify(err));
  });
}
```

**This is the cancel handling the select path is missing** - the same author, the
same picker family, twenty lines apart. The empty-result check returns quietly
and the `.catch` keeps a real failure off the unhandled-rejection path. Use it
as the template for `selectImageFromDocument`.

`newFileNames` only *suggests* the name; the user may rename in the dialog, which
is why the copy targets `documentSaveResult[0]` rather than a path the app
built. The toast is a hardcoded Chinese literal here, unlike the rest of the
sample's strings - it should be a `$r('app.string....')` resource.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions` at all, which is the
point: `DocumentViewPicker` runs in the system picker process and returns URIs
already granted to this app. Reading user files without a media permission is
the intended modern pattern.

Note that the conversion output also lands in `filesDir` and is never cleaned
up - every conversion leaves both a copy of the source and the converted file in
the sandbox.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- Decoding supports JPEG, PNG, GIF, WebP, BMP, SVG, ICO, DNG and HEIF; encoding
  supports **JPEG, WebP, PNG and HEIF only**, and HEIF encoding is
  device-dependent. `OUTPUT_FORMATS` offers `heic` unconditionally, so on a
  device without the encoder the conversion fails inside `packToFile` and the
  save dialog still opens on a zero-byte file.
- `getImageFormat` compares the *source extension* against the output list, so a
  `.jpg` source still offers `jpeg` (and vice versa) - two chips for one encoder.
- The whole source file is read into a single `ArrayBuffer`; there is no
  streaming path for very large images.
- Only the first home tab has content. `onContentWillChange` returns `false` for
  every other index, so the remaining four tabs are unreachable by design.

## Pitfalls

- **`HW-18-0005`** (B/low, confirmed): systematic - `image.createImageSource()`
  is called per conversion and `release()` appears nowhere in the project. Native
  decoder memory accumulates for the process lifetime. Fix: release the source in
  a `finally` once packing has finished. Recorded across eight photography files.
- **`HW-18-0024`** (B/low, confirmed): systematic picker-cancel path -
  `selectResult[0]` is indexed unconditionally, so `url.lastIndexOf` throws on
  cancel and the rethrown error reaches a `.catch`-less `async onClick` in
  `MainPage`. Fix: guard on `length === 0` and return an empty `IImageInfo`. Six
  samples in this industry share the defect; `ImageRotateAndFlip` has the
  correct guard.
- **Both utility `catch` blocks discard the original error.**
  `imageToImageSource` and `selectImageFromDocument` each throw a bare
  `new Error('Failed to ...')`, dropping the `BusinessError` code that would say
  whether the file was unreadable, the format unsupported, or the picker
  cancelled. Rethrow the original, or return a typed result.
- **The converted file is written before the user agrees to save it.** The
  convert button packs into `filesDir` and only then opens the save dialog, so a
  cancelled save leaves the artefact behind with no cleanup.
- **The success toast is a bare Chinese string literal** (`成功保存至本地`) in
  code that otherwise uses string resources - it will not localise.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-image-f.md` - `image.createImageSource`, `createImagePacker`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-f
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagesource.md` - `ImageSource` and `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagesource
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagepacker.md` - `packToFile`, `PackingOption`, supported encoders
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-guides/02_media/image-decoding.md` - the decode-then-release lifecycle
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-decoding
- `documentation/harmonyos-guides/05_media/image-encoding.md` - format and quality selection
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-encoding
- `documentation/harmonyos-references/02_application-framework/js-apis-file-picker.md` - `DocumentViewPicker.select` / `.save`, `DocumentSaveOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-picker
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `openSync`, `statSync`, `readSync`, `copyFileSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `NavPathStack.pushPath` and `NavDestination.onReady`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `PHOTO-09` (CompressImages) - the same encoder used for size reduction, with a correct packer release
