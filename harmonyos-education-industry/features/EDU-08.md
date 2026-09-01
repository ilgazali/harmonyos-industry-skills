---
id: EDU-08
title: PDF to one long image - composite every page into a single PixelMap and save it with a SaveButton
industry: 04_education
doc: huawei_industry_tree/04_education/docs/08_pdf_to_long_image.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/pdf_to_long_image-0000002349147473
sample: huawei_industry_tree/04_education/downloads/Pdf2LongImage.zip
kits: ["@kit.PDFKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.CoreFileKit", "@kit.AbilityKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: [PdfView, "pdfViewManager.PdfController", "pdfService.PdfDocument", loadDocument, getPageCount, getPage, getPagePixelMap, "pdfService.PageFit", "pdfService.ParseResult", "image.createPixelMap", "image.InitializationOptions", readPixelsToBuffer, readPixelsToBufferSync, writePixels, "image.PositionArea", "image.createImagePacker", packToData, "PixelMap.release", SaveButton, SaveDescription, SaveButtonOnClickResult, "photoAccessHelper.getPhotoAccessHelper", createAsset, "resourceManager.getRawFileContentSync", fileIo]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-04-0048, HW-04-0049, HW-04-0050, HW-04-0051, HW-04-0052, HW-04-0053, HW-04-0054, HW-04-0055, HW-04-0056, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card when you need to **turn a multi-page PDF into one tall image** -
an exam paper or a set of lecture slides the student can scroll, screenshot and
share as a single picture.

Note what this is *not*. PDF Kit already has `convertToImage(path, format)`,
which writes **one image per page** into a folder. If per-page files are
acceptable, use that and stop here. This card is for the case where the output
must be a **single** image, which the Kit does not provide - so you rasterise
each page to a `PixelMap` and blit them all into one oversized `PixelMap` at
increasing y offsets.

The two techniques worth taking away are the `PositionArea`/`writePixels` blit
and the `SaveButton` save path that needs no permission at all.

## Feature checklist

- A list of PDFs bundled with the app; tapping one opens a preview.
- The preview is a real `PdfView`, fit to width.
- A 转换长图 button in the title bar; while converting it becomes a spinner with
  a 转换中 caption.
- Conversion produces one image as tall as all pages combined and as wide as the
  widest page.
- The result page previews the long image and offers 保存到图库.
- Saving needs no runtime permission request and no permission declaration.

## Architecture

One `entry` module, three pages, no view model.

```
entry/src/main/ets
├── common/CommonConstants.ets        every dimension and font size
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── model/FileList.ets                FILE_DATA: the three bundled papers
└── pages
    ├── HomePage.ets                  @Entry - the file list, Navigation host
    ├── PdfPage.ets                   PdfView preview + the conversion
    └── SavePage.ets                  long-image preview + SaveButton
```

The documented tree matches the zip.

**Two independent PDF objects, and both are needed.** `PdfPage` holds a
`pdfViewManager.PdfController` *and* a `pdfService.PdfDocument`, and calls
`loadDocument` on each with the same path:

- the **controller** drives the on-screen `PdfView`;
- the **document** is the headless model that exposes `getPageCount`,
  `getPage(i)` and `getPagePixelMap()`.

They do not share state. Loading only the controller gives you a preview you
cannot read pixels from; loading only the document gives you pixels with no
preview.

**Page hand-off is through `AppStorage`.** The composite `PixelMap` is parked
under the key `pixelMap` and `SavePage` reads it back. That is the pragmatic
choice for a large native object - it must not be serialised through the
navigation parameter - but it also means nothing owns it, which is why nothing
releases it (`HW-04-0051`).

**PDFs are shipped as rawfiles and copied to the sandbox on first open.** PDF
Kit's `loadDocument` takes a filesystem path, and a rawfile is not one; so
`aboutToAppear` checks `fileIo.accessSync(filesDir + '/' + name)` and, if
absent, reads the rawfile and writes it out. The read uses the wrong path form
(`HW-04-0048`), which is what breaks the sample on a clean install.

## Implementation steps

1. **Copy the PDF into the sandbox** if it is not already there:
   `resourceManager.getRawFileContentSync(fileName)` - the path is relative to
   `resources/rawfile`, with **no** `rawfile/` prefix - then `openSync` with
   `WRITE_ONLY | CREATE | TRUNC`, `writeSync(fd, content.buffer)`, `closeSync`.
2. **Load the same path into both objects** and keep the `ParseResult` from
   `pdfDocument.loadDocument`; every later step is gated on
   `ParseResult.PARSE_SUCCESS`.
3. **Measure first, allocate once.** Loop the pages collecting
   `getWidth()`/`getHeight()`, sum the heights, take the max width, and allocate
   one `ArrayBuffer` of `maxWidth * totalHeight * 4`.
4. **Create the composite `PixelMap`** from that buffer with
   `image.createPixelMap(buffer, { editable: true, pixelFormat: RGBA_8888,
   size: { width: maxWidth, height: totalHeight } })`. `editable: true` is
   mandatory - `writePixels` fails on a read-only map.
5. **Blit each page in.** Get its `PixelMap`, read its pixels into a per-page
   buffer with **`readPixelsToBufferSync`** (or `await` the async form -
   `HW-04-0050`), then `writePixels` with a `PositionArea` whose `region.y` is
   the running offset and whose `stride` is `pageWidth * 4`.
6. **Release each page `PixelMap`** at the end of the iteration, and release the
   previous composite before storing a new one (`HW-04-0051`).
7. **Wrap the whole handler in `try`/`finally`** and clear the converting flag
   in the `finally` (`HW-04-0053`).
8. **Run it off the UI thread.** A page rasterise per page in a click handler
   blocks the very spinner it puts up (`HW-04-0056`).
9. **Pack and save.** `image.createImagePacker().packToData(pixelMap, { format:
   'image/png', quality: 98 })`, **awaited**, then a `SaveButton` whose
   `onClick` calls `photoAccessHelper.createAsset` within ten seconds of the tap.
10. **Declare no permission.** The `SaveButton` is the documented alternative to
    `WRITE_IMAGEVIDEO` (`HW-04-0049`).

## Verified snippets

All snippets are from `Pdf2LongImage.zip`. Corrected forms are marked.

**Rawfile to sandbox, then load both objects — `entry/src/main/ets/pages/PdfPage.ets`** (corrected, see `HW-04-0048`)

```typescript
import { pdfService, PdfView, pdfViewManager } from '@kit.PDFKit';
import { fileIo } from '@kit.CoreFileKit';

private controller: pdfViewManager.PdfController = new pdfViewManager.PdfController();
private pdfDocument: pdfService.PdfDocument = new pdfService.PdfDocument();
private loadResult: pdfService.ParseResult = pdfService.ParseResult.PARSE_ERROR_FORMAT;

async aboutToAppear(): Promise<void> {
  const context = this.getUIContext().getHostContext() as common.UIAbilityContext;
  const fileName = AppStorage.get('fileName') as string;
  const filePath: string = context.filesDir + '/' + fileName;

  if (!fileIo.accessSync(filePath)) {
    // FIX: sample passes 'rawfile/' + fileName, which resolves to resources/rawfile/rawfile/...
    const content: Uint8Array = context.resourceManager.getRawFileContentSync(fileName);
    const fd = fileIo.openSync(filePath,
      fileIo.OpenMode.WRITE_ONLY | fileIo.OpenMode.CREATE | fileIo.OpenMode.TRUNC);
    fileIo.writeSync(fd.fd, content.buffer);
    fileIo.closeSync(fd.fd);
  }

  await this.controller.loadDocument(filePath);          // drives the PdfView
  this.loadResult = this.pdfDocument.loadDocument(filePath);  // drives getPage/getPagePixelMap
}
```

**`getRawFileContentSync`'s argument is already rooted at `resources/rawfile`.**
The reference's own example passes `"test.txt"`, and error `9001005 Invalid
relative path` is what a prefixed path produces. `EnglishPractice` in this same
industry calls it correctly (`'singlequestion.json'`); this sample does not, and
because the call sits outside a `try`, the page throws on first open.

**`TRUNC` matters**: without it, re-copying over a shorter file leaves the tail
of the previous one and the PDF fails to parse.

**Composite the pages — same file** (corrected, see `HW-04-0050`, `HW-04-0051`, `HW-04-0053`, `HW-04-0054`)

```typescript
interface HeightAndWidth { width: number; height: number; }

async convert(): Promise<void> {
  if (this.loadResult !== pdfService.ParseResult.PARSE_SUCCESS) {
    return;
  }
  this.isConverting = true;
  try {                                                   // FIX: sample has no try/finally
    const pageCount = this.pdfDocument.getPageCount();

    // pass 1 - measure every page, allocate exactly once
    const pageDimensions: Array<HeightAndWidth> = [];
    for (let i = 0; i < pageCount; i++) {
      const page = this.pdfDocument.getPage(i);
      pageDimensions.push({ width: page.getWidth(), height: page.getHeight() });
    }
    const totalHeight = pageDimensions.reduce((sum, d) => sum + d.height, 0);
    const maxPageWidth = Math.max(...pageDimensions.map(d => d.width));

    const combineBuffer = new ArrayBuffer(maxPageWidth * totalHeight * 4);   // RGBA = 4 bytes
    const opts: image.InitializationOptions = {
      editable: true,                                     // required for writePixels
      pixelFormat: image.PixelMapFormat.RGBA_8888,
      size: { width: maxPageWidth, height: totalHeight }
    };
    const newPixelMap = await image.createPixelMap(combineBuffer, opts);

    // pass 2 - blit each page at the running y offset
    let currentY = 0;
    for (let i = 0; i < pageCount; i++) {
      const page = this.pdfDocument.getPage(i);
      const singlePixel = page.getPagePixelMap();
      const pageWidth = pageDimensions[i].width;
      const pageHeight = pageDimensions[i].height;
      // FIX: sample builds a DecodingOptions object here purely to read this size back
      const pagePixelBuffer = new ArrayBuffer(pageWidth * pageHeight * 4);
      try {
        singlePixel.readPixelsToBufferSync(pagePixelBuffer);   // FIX: sample drops the promise
        const area: image.PositionArea = {
          pixels: pagePixelBuffer,
          offset: 0,
          stride: pageWidth * 4,                               // bytes per row of the SOURCE
          region: { size: { width: pageWidth, height: pageHeight }, x: 0, y: currentY }
        };
        await newPixelMap.writePixels(area);
        currentY += pageHeight;
      } catch (error) {
        hilog.error(0x0000, 'Pdf2LongImage', `page ${i + 1} failed: ${error}`);
      } finally {
        await singlePixel.release();                           // FIX: absent in the sample
      }
    }

    const previous: PixelMap | undefined = AppStorage.get('pixelMap');
    await previous?.release();                                 // FIX: absent in the sample
    AppStorage.setOrCreate('pixelMap', newPixelMap);
    this.pageInfos.pushPathByName('SavePage', '');
  } finally {
    this.isConverting = false;                                 // FIX: sample only clears on success
  }
}
```

**`PositionArea` is the blit primitive, and `stride` is the part to get right.**
`region` says *where in the destination* the rectangle lands; `stride` says how
many bytes make up one row *in the source buffer*. They are about different
images. Here every page buffer is exactly its own page's width, so
`pageWidth * 4` is correct - but the destination is `maxPageWidth` wide, so a
narrower page leaves the right-hand strip at whatever
`image.createPixelMap` left in the fresh buffer.

**Measure-then-allocate is why this is two loops.** You cannot know the height
of the composite without walking every page, and a `PixelMap` cannot be resized
after creation, so the first loop exists purely to size the second's target.

**`readPixelsToBuffer` returns a promise.** Dropped, the copy is still in flight
when `writePixels` reads the same buffer, and the page lands as a black band.
`readPixelsToBufferSync` is the better fit here since the loop is sequential
anyway.

**Save with no permission — `entry/src/main/ets/pages/SavePage.ets`** (corrected, see `HW-04-0052`)

```typescript
@State pixelMap: PixelMap | undefined = undefined;
@State ready: boolean = false;                  // FIX: sample has no readiness gate
private arrayBuffer: ArrayBuffer | undefined = undefined;

async aboutToAppear(): Promise<void> {
  this.pixelMap = AppStorage.get('pixelMap');
  if (!this.pixelMap) {
    return;
  }
  const packer: image.ImagePacker = image.createImagePacker();
  const packOpts: image.PackingOption = { format: 'image/png', quality: 98 };
  try {
    this.arrayBuffer = await packer.packToData(this.pixelMap, packOpts);  // FIX: sample drops this
    this.ready = true;
  } catch (err) {
    hilog.error(0x0000, 'SavePage', `packToData failed: ${err}`);
  }
}

SaveButton({ text: SaveDescription.SAVE_TO_GALLERY })
  .enabled(this.ready)
  .onClick(async (_event: ClickEvent, result: SaveButtonOnClickResult) => {
    if (result !== SaveButtonOnClickResult.SUCCESS) {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.save_failed') });
      return;
    }
    const context = this.getUIContext().getHostContext();
    const helper = photoAccessHelper.getPhotoAccessHelper(context);
    // createAsset must be called within 10 s of the tap - the grant expires, the fd does not
    const uri = await helper.createAsset(photoAccessHelper.PhotoType.IMAGE, 'png');
    const file = await fileIo.open(uri, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
    await fileIo.write(file.fd, this.arrayBuffer);
    await fileIo.close(file.fd);
    this.pageInfos.clear();
    this.getUIContext().getPromptAction().showToast({ message: $r('app.string.save_successfully') });
  })
```

**The `SaveButton` grants the write, not a permission.** Tapping it hands the
app a short-lived authorisation, and `result === SaveButtonOnClickResult.SUCCESS`
is the signal that the grant arrived. The restricted-permissions guide names
this exact control as the "recommended solution without the permission" for
`WRITE_IMAGEVIDEO` - so declaring that permission alongside it, as the sample
does, is redundant and unobtainable at normal APL.

**The ten-second window applies to `createAsset`, not to the write.** Once the
asset exists and you hold its fd, writing can take as long as it needs - which
matters here, because a long image is a large PNG.

## Permissions & config

**None.** The corrected `entry/src/main/module.json5` has no `requestPermissions`
block at all:

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "pages": "$profile:main_pages"
    // FIX: the sample declares ohos.permission.WRITE_IMAGEVIDEO here - see HW-04-0049
  }
}
```

- `ohos.permission.WRITE_IMAGEVIDEO` is **restricted**, at `system_basic` level:
  a normal-APL application cannot obtain it, and the guide's recommended
  alternative is precisely the `SaveButton` this sample already uses.
- The three papers live at `entry/src/main/resources/rawfile/试卷1.pdf` … `3.pdf`
  and are copied into `filesDir` on first open.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The composite is a single allocation of `maxWidth * totalHeight * 4`
  bytes.** That grows linearly with page count and quadratically with
  resolution; a long document will fail at `image.createPixelMap` rather than
  degrade. There is no tiling and no cap on page count.
- Pages narrower than the widest page leave an uninitialised strip on the right
  of the composite - the sample never clears the buffer.
- The whole conversion is synchronous with respect to the UI thread
  (`HW-04-0056`).
- `PdfDocument` is never closed and no `PixelMap` is released (`HW-04-0051`).
- The three bundled PDFs are byte-identical (87211 bytes each); only the
  displayed name differs.
- If per-page files are acceptable, `pdfDocument.convertToImage(outputPath,
  pdfService.ImageFormat.PNG)` does the whole job in one call and none of this
  code is needed.

## Pitfalls

- **`HW-04-0048` — `getRawFileContentSync('rawfile/' + name)` is wrong** and
  throws `9001005`. The argument is already relative to `resources/rawfile`.
  Uncaught, in `aboutToAppear`, on first open of every document - the sample
  does not work on a clean install. `EnglishPractice` in the same industry calls
  the same API correctly.
- **`HW-04-0049` — `WRITE_IMAGEVIDEO` is declared and cannot be granted.** It is
  restricted at `system_basic` level, and the `SaveButton` in the same sample is
  its documented replacement.
- **`HW-04-0050` — `readPixelsToBuffer` is not awaited,** so `writePixels` reads
  a buffer that is still being filled. Use `readPixelsToBufferSync`.
- **`HW-04-0051` — nothing releases any `PixelMap`.** One per page plus the
  composite, all native memory, all retained; converting a second document
  orphans the first composite in `AppStorage`.
- **`HW-04-0052` — `packToData` is not awaited and has no catch,** so an early
  tap on Save creates a gallery asset and writes `undefined` into it, then
  reports success.
- **`HW-04-0053` — a failure before the page loop leaves `isConverting` true**
  and the button permanently replaced by a spinner. `image.createPixelMap` on an
  oversized composite is exactly the call that fails there.
- **`HW-04-0054` — a `DecodingOptions` object is built per page and consumed by
  nothing,** existing only so its `desiredSize` can be read back. Its
  `BGRA_8888` is never applied and contradicts the composite's `RGBA_8888`.
- **`HW-04-0055` — the document's snippets do not compile and disagree with the
  code:** an unclosed `for`, no `try`/`catch`, `showScroll: true` versus
  `false`, `SaveDescription.SAVE` versus `SAVE_TO_GALLERY`.
- **`HW-04-0056` — the conversion runs on the UI thread,** which starves the
  spinner it puts up to indicate progress.
- **`editable: true` is not optional** in the composite's
  `InitializationOptions`; `writePixels` on a read-only `PixelMap` fails.

## References

- `documentation/harmonyos-guides/07_application-services/pdf-get-img.md` - `getPagePixelMap`, `getCustomPagePixelMap`, `getAreaPixelMapWithOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/pdf-get-img
- `documentation/harmonyos-guides/07_application-services/pdf-convert-img.md` - `convertToImage`, the per-page alternative to this whole card
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/pdf-convert-img
- `documentation/harmonyos-references/04_media/arkts-apis-image-pixelmap.md` - `writePixels`, `PositionArea`, `readPixelsToBuffer(Sync)`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-pixelmap
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFileContentSync` and its rawfile-relative path
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - `WRITE_IMAGEVIDEO` and its no-permission alternative
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `documentation/harmonyos-guides/04_system/app-permission-group-list.md` - the Files/media permission group and the picker/SaveButton alternatives
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/app-permission-group-list
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` - `SaveButton`, `SaveDescription`, `SaveButtonOnClickResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `EDU-07` - the correct `getRawFileContentSync` call in the same industry
- `EDU-09` - importing a PDF from the file picker rather than from a rawfile
