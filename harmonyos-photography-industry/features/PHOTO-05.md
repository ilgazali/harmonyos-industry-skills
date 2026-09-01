---
id: PHOTO-05
title: Template photo collage - decode each picture to its slot size and blit it into one PixelMap
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/05_image_stitch.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_stitch-0000002287473193
sample: huawei_industry_tree/18_photography/downloads/ImageStitch.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [display, fileIo, hilog, image, photoAccessHelper, window, createPixelMapSync, createPixelMapSync, readPixelsToBufferSync, writePixelsSync, PositionArea, DecodingOptions, InitializationOptions, PhotoViewPicker, SaveButton, createAsset, createImagePacker, packToData]
permissions: [ohos.permission.WRITE_IMAGEVIDEO]
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0028, HW-18-0004, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when you need to **combine several pictures into one new image
inside the app**, on a fixed set of layout templates - a collage, a
before/after pair, a photo-strip, a receipt made of three shots. The user picks
two or three images, taps a template thumbnail, and the app produces a single
new `PixelMap` that can be previewed and saved.

The pattern is: allocate one blank `PixelMap` of the final canvas size, decode
each source image **straight to the pixel size of the slot it will occupy**
using `DecodingOptions.desiredSize`, then copy it into place with
`writePixelsSync` and a `PositionArea`. There is no `Canvas` component, no
offscreen render, no layout pass - the scaler runs inside the image decoder and
the composition is one buffer copy per slot.

It generalises to any raster composition where the geometry is known in
advance: watermarking, contact sheets, splitting a panorama, or laying out a
QR code next to a photo. What it is *not* good for is free composition where
the user drags and resizes pieces - there the sibling `PictureSticker` sample
(`PHOTO-10`) and a real `Canvas` are the right tools.

## Feature checklist

- A full-screen black page with a large preview area on top, a template strip
  in the middle and a three-button action row at the bottom.
- Tapping the plus icon opens the system photo picker limited to images, with
  at most three selectable.
- Selecting exactly two images shows the two-image template chooser; exactly
  three shows the three-image chooser; anything else raises a toast and leaves
  the preview untouched.
- Each template thumbnail recomposes the preview immediately: two images side
  by side or stacked; three images as one full-height left panel plus two
  quarters, or one full-width top panel plus two quarters.
- The stitched result renders in the preview `Image` as a `PixelMap`, not a
  file path.
- The `SaveButton` encodes the result to JPEG and writes it into the gallery,
  toasting on success; pressing it with nothing stitched toasts instead.
- The 取消 (cancel) button is a demo stub that only toasts.

## Architecture

One `entry` module. Three stencil functions, one utility file, two chooser
views, one page.

```
entry/src/main/ets
├── common/Constants.ets                 the canvas maths: 500, 800, 250, 400, 1000
├── entryability/EntryAbility.ets        full-screen layout, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages/Index.ets                      @Entry: preview + chooser slot + action row
├── utils/
│   ├── TwoImageStencil.ets              2 images, side by side (isH) or stacked
│   ├── ThreeImageStencilFirst.ets       3 images: full-height left + two right quarters
│   ├── ThreeImageStencilSec.ets         3 images: full-width top + two bottom quarters
│   └── Utils.ets                        picker, decode helpers, ImagePacker, gallery save
└── views/
    ├── TwoImagePopBoxView.ets           template chooser shown for 2 images
    └── ThreeImagePopBoxView.ets         template chooser shown for 3 images
```

The documented 工程目录 matches the zip file for file, including the
`EntryBackupAbility.ets` spelling that three sibling docs in this industry get
wrong (`HW-18-0002`).

**The design decision worth copying** is that a stencil is a *pure function*,
not a component: `TwoImageStencil(images, isH)` takes the picked URIs and
returns `image.PixelMap | null`. It owns no state, touches no UI, and is
called identically from the page (right after the pick) and from the chooser
view (when the user switches template). That is why the same four lines in
`Index.ets` and in `TwoImagePopBoxView.ets` can both drive the preview through
one `@Provide joinImagePixelMap` / `@Consume` pair - the page provides the
slot, whoever recomputes fills it.

**One structural detail worth avoiding**: `Index.aboutToAppear` measures the
display and stores `areaWidth`/`areaHeight`, which are passed down into
`ThreeImagePopBoxView` as `@Link`s - and never read there. The canvas size is
in fact the hardcoded 500x800 in `Constants.ets`. The measurement is dead
code that suggests a responsiveness the sample does not have.

## Implementation steps

1. **Pick with `PhotoViewPicker`, not a permission.** `MIMEType =
   IMAGE_TYPE`, `maxSelectNumber = 3`. The returned URIs carry read access,
   so no album permission is needed - which is exactly why the declared
   `WRITE_IMAGEVIDEO` in `module.json5` is wrong here (`HW-18-0004`).
2. **Branch on `photoUris.length`, never on `undefined`.** A cancelled pick
   resolves to an empty array; this sample's `length === 2 / === 3 / else`
   chain handles it correctly and is the reference form for the cancel bug
   that hits six sibling samples (`HW-18-0024`).
3. **Fix the canvas first.** Create `InitializationOptions` with
   `pixelFormat: BGRA_8888` and `size: {500, 800}`, back it with a zero-filled
   `ArrayBuffer` of `width * height * 4`, and build it with
   `image.createPixelMapSync`.
4. **Decode each source to its slot size** via `DecodingOptions.desiredSize`,
   with `desiredPixelFormat` equal to the canvas format. Mixing formats here
   is the single easiest way to get garbage output - the sample's own comment
   says 注意上下格式统一 (keep the formats consistent above and below).
5. **Read the slot out with `readPixelsToBufferSync`** and write it in with
   `writePixelsSync({ pixels, offset, stride, region })`.
6. **Compute `stride` as the region width times 4**, not the canvas width -
   the buffer being written is the *sub-image's* buffer, so its row length is
   the sub-image's. All four stencil branches in this sample get this right;
   it is the first thing to check when a collage comes out sheared.
7. **Never route a `PixelMap` input through `createImageSource`**
   (`HW-18-0028`). Keep decoded pixel maps in their own list, or encode them
   to PNG with an `ImagePacker` first.
8. **Save inside the `SaveButton` click window.** `createAsset` is authorised
   for five seconds after `SaveButtonOnClickResult.SUCCESS`; open the returned
   URI in that window, then write at leisure.

## Verified snippets

All snippets are from `ImageStitch.zip`. Corrected forms are marked.

**Pick, dispatch, save - `entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
Image($r('app.media.ic_plus'))
  .onClick(async () => {
    this.joinImages = await selectImages(3);
    if (this.joinImages.length === 2) {
      if (this.isSelected) {
        this.joinImagePixelMap = TwoImageStencil(this.joinImages, false);   // stacked
      } else {
        this.joinImagePixelMap = TwoImageStencil(this.joinImages);          // side by side
      }
    } else if (this.joinImages.length === 3) {
      if (this.isSelected) {
        this.joinImagePixelMap = ThreeImageStencilFirst(this.joinImages);
      } else {
        this.joinImagePixelMap = ThreeImageStencilSec(this.joinImages);
      }
    } else {
      this.promptAction.showToast({ message: $r('app.string.index_prompt_tips') });
    }
  });

SaveButton()
  .onClick(async (event: ClickEvent, result: SaveButtonOnClickResult) => {
    if (!this.joinImagePixelMap) {
      this.promptAction.showToast({ message: $r('app.string.index_prompt_splicing') });
      return;
    }
    if (result === SaveButtonOnClickResult.SUCCESS) {
      savePixelMapToGalleryBySaveButton(this.uiContext, this, this.joinImagePixelMap);
    } else {
      this.promptAction.showToast({ message: $r('app.string.index_prompt_Hints') });
    }
  });
```

**The `else` branch is the whole cancel story.** `photoViewPicker.select()`
resolves with `photoUris: []` when the user backs out, so `length` lands in
neither the 2 nor the 3 case and the user gets 请选择2-3张图片 instead of an
unhandled rejection. Six other samples in this industry index `[0]` on that
same empty array and crash (`HW-18-0024`); this one is the pattern to copy.

The `SaveButton` guard reads `result === SaveButtonOnClickResult.SUCCESS`
before doing anything. That check is not decoration: the security component
grants the temporary `createAsset` capability only when the click was a real,
unspoofed user tap, and any other result means the app has no write authority
at all for the next five seconds.

**The stitch loop - `entry/src/main/ets/utils/TwoImageStencil.ets`** (corrected, see `HW-18-0028`)

```typescript
export function TwoImageStencil(images: Array<string> | Array<image.PixelMap>,
  isH = true): image.PixelMap | null {
  let imageSourceList: Array<image.ImageSource | undefined> = [];
  let pixelMapList: Array<image.PixelMap | undefined> = [];
  try {
    if (images.length < 2) {
      return null;
    }
    for (let x = 0; x < images.length; x++) {
      if (typeof images[x] === 'string') {
        let file = fileIo.openSync(String(images[x]), fileIo.OpenMode.READ_ONLY);
        let buffer = new ArrayBuffer(fileIo.statSync(file.fd).size);
        fileIo.readSync(file.fd, buffer);          // encoded bytes: decodable
        fileIo.closeSync(file);
        imageSourceList[x] = image.createImageSource(buffer);
      } else {
        // FIX: the sample does readPixelsToBuffer(buffer) un-awaited and then
        // createImageSource(buffer) - raw BGRA can never be decoded as a file.
        pixelMapList[x] = images[x] as image.PixelMap;
      }
    }

    let combineOpts: image.InitializationOptions = {
      alphaType: 0,
      editable: true,
      // 注意上下格式统一 - the canvas and every slot must share a pixel format
      pixelFormat: image.PixelMapFormat.BGRA_8888,
      size: { width: Constants.NUMBER_500, height: Constants.NUMBER_800 }
    };
    let combineColor = new ArrayBuffer(combineOpts.size.width * combineOpts.size.height * 4);
    let newPixelMap = image.createPixelMapSync(combineColor, combineOpts);

    for (let x = 0; x < images.length; x++) {
      let singleOpts: image.DecodingOptions = {
        editable: true,
        desiredPixelFormat: image.PixelMapFormat.BGRA_8888,
        desiredSize: isH ? { width: Constants.NUMBER_500 * Constants.NUMBER_ONE_HALF, height: Constants.NUMBER_800 } :
          { width: Constants.NUMBER_500, height: Constants.NUMBER_800 * Constants.NUMBER_ONE_HALF }
      };
      let singleColor = new ArrayBuffer(singleOpts.desiredSize!.width * singleOpts.desiredSize!.height * 4);
      let source = imageSourceList[x];
      let singlePixelMap: image.PixelMap =
        source !== undefined ? source.createPixelMapSync(singleOpts) : pixelMapList[x]!;
      singlePixelMap.readPixelsToBufferSync(singleColor);

      let area: image.PositionArea = {
        pixels: singleColor,
        offset: Constants.NUMBER_0,
        stride: isH ? Constants.NUMBER_1000 : Constants.NUMBER_500 * 4,
        region: {
          size: isH ? { height: Constants.NUMBER_800, width: Constants.NUMBER_250 } :
            { height: Constants.NUMBER_400, width: Constants.NUMBER_500 },
          x: isH ? (x === Constants.NUMBER_0 ? Constants.NUMBER_0 : Constants.NUMBER_250) : Constants.NUMBER_0,
          y: isH ? Constants.NUMBER_0 : (x === Constants.NUMBER_0 ? Constants.NUMBER_0 : Constants.NUMBER_400)
        }
      };
      newPixelMap.writePixelsSync(area);
    }
    return newPixelMap;                            // FIX: the sample returns this from inside `if (true)`
  } catch (err) {
    hilog.error(0x0000, 'JOIN_IMAGES', 'PictureJoinTogether join error: ' + JSON.stringify(err));
    return null;
  }
}
```

**Three numbers carry the geometry.** `desiredSize` is the size the decoder
will produce - decoding straight to 250x800 rather than decoding full size and
scaling afterwards is what keeps a three-image collage from allocating three
full-resolution buffers. `stride` is the row length **of the source buffer**,
so it is `region.width * 4` (1000 for a 250-wide half, 2000 for a 500-wide
half), never the canvas width. `region.x` / `region.y` are the destination
offsets into the canvas. Get `stride` wrong and the slot comes out sheared
diagonally, which is the classic symptom.

The shipped file also wraps the whole blit loop in a literal `if (true) {`,
which leaves the function with a code path that returns nothing while its
signature promises `PixelMap | null`. Deleting the wrapper is part of the fix.

**Encode and save - `entry/src/main/ets/utils/Utils.ets`** (as shipped)

```typescript
export async function packing(pixelMap: image.PixelMap, format: string, quality = 100) {
  const imagePackerApi = image.createImagePacker();
  let packOpts: image.PackingOption = { format, quality };
  let arrayBuffer = await imagePackerApi.packToData(pixelMap, packOpts);
  return arrayBuffer;                              // no imagePackerApi.release() - see HW-18-0036
}

export async function savePhotoToGalleryBySaveButton(uiContext: UIContext, _component: object,
  imgArrayBuffer: ArrayBuffer, saveType = 'jpeg', isShowToast = true) {
  const context = uiContext.getHostContext();
  let helper = photoAccessHelper.getPhotoAccessHelper(context);
  try {
    // onClick触发后5秒内通过createAsset接口创建图片文件，5秒后createAsset权限收回。
    let uri = await helper.createAsset(photoAccessHelper.PhotoType.IMAGE, saveType);
    // 使用uri打开文件，可以持续写入内容，写入过程不受时间限制
    let file = await fileIo.open(uri, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
    await fileIo.write(file.fd, imgArrayBuffer);
    await fileIo.close(file.fd);
    if (isShowToast) {
      uiContext.getPromptAction().showToast({ message: $r('app.string.utils_saveimage') });
    }
  } catch (error) {
    const err: BusinessError = error as BusinessError;
    hilog.error(0x0000, 'SAVE_PHOTO', `Failed to save photo. Code is ${err.code}, message is ${err.message}`);
  }
}
```

**The two comments in this function are the contract.** `createAsset` must be
called within five seconds of the `SaveButton` click; the *write* into the URI
it returns has no time limit. So the correct shape is: create the asset first
(cheap, inside the window), then open and write - which is what this code
does, and it is why `packing()` runs before `savePhotoToGalleryBySaveButton` is
entered rather than between `createAsset` and `open`. Encoding a 500x800 JPEG
takes tens of milliseconds; encoding a 48MP one might not fit in five seconds.

The `ImagePacker` is created per save and never released. `CompressImages` in
this same industry releases it correctly; eight other samples do not
(`HW-18-0036`). Add `imagePackerApi.release()` in a `finally`.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.WRITE_IMAGEVIDEO",     // should not be here
    "reason": "$string:reason",
    "usedScene": { "abilities": ["EntryAbility"], "when": "inuse" }
  }
]
```

`WRITE_IMAGEVIDEO` is a restricted (ACL) permission that ordinary apps cannot
ship with, and this sample never requests it at runtime - it reads through
`PhotoViewPicker` and writes through `SaveButton`, both of which are
permission-free by design. Delete the entry (`HW-18-0004`); the sample works
without it, and shipping it earns an app-review rejection.

`deviceTypes` is `phone`, `tablet`, `2in1`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- **The canvas is a fixed 500x800.** Every source image is squashed to its slot
  aspect ratio - there is no letterboxing, no crop-to-fill and no respect for
  portrait vs landscape inputs. For real use, compute the canvas from the
  inputs and pick a fit strategy before touching `desiredSize`.
- The stencils are synchronous and run on the UI thread. Three full-size
  decodes plus three buffer copies at 500x800 are fast; at 4000x3000 they are
  not. Move them to a worker or use the async `createPixelMap` variants.
- `createPixelMapSync` over a zero-filled buffer gives a transparent-black
  canvas, so any slot the templates leave uncovered stays black. The
  three-image templates cover the canvas exactly; a fourth template would need
  checking.
- The `PixelMap` input overload (`Array<image.PixelMap>`) is dead in the
  shipped code (`HW-18-0028`); only the URI overload is reachable from the UI.
- No `PixelMap` is ever released. The preview holds one and every template tap
  allocates another.

## Pitfalls

- **`HW-18-0028`** (B/low, confirmed): all three stencil utilities call
  `pixelMap1.readPixelsToBuffer(buffer)` **without `await`** and then feed the
  raw BGRA buffer to `image.createImageSource`, which expects encoded file
  bytes - so the `PixelMap` input branch can never decode, and the buffer may
  not even be filled when it is read. `TwoImageStencil` additionally wraps the
  blit loop in a dead `if (true) {`. Fix: use the pixel maps directly (or pack
  them to PNG first) and drop the wrapper.
- **`HW-18-0004`** (D/medium, confirmed): systematic - nine photography
  samples, this one included, declare the restricted `READ`/`WRITE_IMAGEVIDEO`
  permissions although their code is built entirely around the
  permission-free `SaveButton` and `PhotoViewPicker` flows and never calls
  `requestPermissionsFromUser` for them. One shared `module.json5` template is
  the evident root cause. Fix: delete the entry.
- **Dead responsive plumbing**: `areaWidth` / `areaHeight` are measured from
  `display.getAllDisplays` in `aboutToAppear` and threaded into
  `ThreeImagePopBoxView` as `@Link`s that nothing reads. Harmless, but it hides
  the fact that the geometry is hardcoded.
- **`ImagePacker` leaked per save** - not enumerated for this sample, but the
  same `createImagePacker()`-without-`release()` shape as `HW-18-0036`.

## References

- `huawei_industry_tree/18_photography/docs/05_image_stitch.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_stitch-0000002287473193
- `documentation/harmonyos-guides/05_media/image-overview.md` - Image Kit overview
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-overview
- `documentation/harmonyos-guides/05_media/image-decoding-arts.md` - `createImageSource`, `DecodingOptions`, `desiredSize`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-decoding-arts
- `documentation/harmonyos-guides/05_media/image-pixelmap-operation.md` - `readPixelsToBufferSync`, `writePixelsSync`, `PositionArea`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-pixelmap-operation
- `documentation/harmonyos-guides/05_media/image-encoding-arts.md` - `ImagePacker`, `packToData` and releasing the packer
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-encoding-arts
- `documentation/harmonyos-guides/05_media/photoaccesshelper-savebutton.md` - the `SaveButton` + `createAsset` five-second window
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-savebutton
- `documentation/harmonyos-guides/05_media/photoaccesshelper-photoviewpicker.md` - the permission-free picker
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `createAsset`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` - `SaveButtonOnClickResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `PHOTO-10` - free-form sticker composition, where a `Canvas` replaces the stencils
- `PHOTO-08` - the other save path in this industry, `showAssetsCreationDialog`
