---
id: PHOTO-28
title: Oil-painting and pencil-sketch filters - per-pixel kernels on a taskpool thread, saved through the creation dialog
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/28_image_filter_processing.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_filter_processing-0000002520461010
sample: huawei_industry_tree/18_photography/downloads/ImageFilterProcessing.zip
kits: ["@kit.AbilityKit", "@kit.ArkTS", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [common, fileIo, fileuri, hilog, image, photoAccessHelper, taskpool, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0080, HW-18-0081, HW-18-0082, HW-18-0083, HW-18-0036, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when you need an **image filter that no built-in effect
provides** and you have to write the pixel loop yourself. The two filters here
are worked examples: an oil-painting effect built from per-pixel colour
histograms, and a pencil sketch built from grayscale, separable Gaussian blur,
a division blend and a paper texture.

The reusable scaffolding is the part around the kernels. A `PixelMap` is read
once into an `ArrayBuffer`, the buffer plus its dimensions travel to a
`@Concurrent` function on a `taskpool` thread, the result buffer comes back and
is turned into a new `PixelMap` with `image.createPixelMapSync`. The UI shows a
`LoadingProgress` overlay for the duration and disables the save button. That
scaffold works for any per-pixel algorithm - a `PixelMap` cannot cross the
concurrency boundary, but a plain buffer can, so the filter code stays pure and
testable.

The save path is worth taking too: `showAssetsCreationDialog` writes to the
gallery with **no permission at all**, because the user picks the destination
in a system dialog. For a one-off save that is strictly better than requesting
WRITE_IMAGEVIDEO.

**Both correctness defects in this card are in the oil-painting kernel**
(`HW-18-0080`, `HW-18-0081`): it crashes on white pixels and swaps red with
blue. Fix both before shipping anything derived from it.

## Feature checklist

- A photo-edit page opens on a bundled `rawfile` image and shows it centred.
- A filter strip offers three thumbnails: original, oil painting, pencil
  sketch; the selected one is outlined in blue.
- Choosing a filter dims the screen behind a spinner and disables the save
  button until the result arrives.
- Choosing "original" restores the untouched `PixelMap` instantly, with no
  task.
- The oil-painting result shows flat, banded colour regions; the pencil result
  shows dark outlines on a lightly textured white ground.
- The save icon opens the system asset-creation dialog and copies the packed
  JPEG into the chosen gallery entry.
- A toast confirms the save.

## Architecture

One `entry` module. The two kernels are plain functions in `utils`, with no UI
or framework dependency at all.

```
entry/src/main/ets
├── constants/StyleConstants.ets      layout numbers only
├── entryability/EntryAbility.ets     full screen + avoid areas -> AppStorage
├── entrybackupability/
├── pages/MainPage.ets                @Entry: the strip, the overlay, the @Concurrent dispatcher
├── utils
│   ├── ImageUtil.ets                 rawfile -> PixelMap, buffer round-trip, packing, gallery save
│   ├── OilPaintingFilterUtil.ets     ColorTank + pixelMapOilPaintingFilter
│   └── PencilSketchFilterUtil.ets    grayscale, separable blur, edge enhance, paper texture
└── viewModel/OptionViewModel.ts      ImageFilterType enum + the ImageInfo transfer object
```

The documented tree lists `viewModel/OptionViewModel.ets`; the zip ships
`OptionViewModel.ts`. Harmless, but it is the one place the two disagree.

**The design decision worth copying** is `ImageInfo` in `OptionViewModel.ts`.
It is a plain class holding `readBuffer`, `pixelFormat`, `width`, `height`,
`radius` and `tank` - nothing else. It is the *only* thing that crosses the
taskpool boundary in either direction, which is what lets one `@Concurrent`
dispatcher serve both filters and lets the filter parameters (`radius` for
brush/line thickness, `tank` for the number of colour buckets) travel with the
data instead of being captured. A `PixelMap` is not serialisable across a
taskpool task; an `ArrayBuffer` is transferred, not copied. Keeping the DTO
this thin is what makes both true.

## Implementation steps

1. **Decode once, keep the original.** `getPixelMapFromRaw` decodes with
   `editable: true` and `desiredPixelFormat: 3` (RGBA_8888); `MainPage` keeps
   `pixelMap` as the pristine source and `pixelMapTemp` as what is displayed,
   so re-filtering never compounds.
2. **Read pixels into a buffer** sized `width * height * 4` with
   `readPixelsToBufferSync`, and carry `pixelFormat`, `width` and `height`
   alongside it.
3. **Run the kernel in a `@Concurrent` function** dispatched by
   `taskpool.execute`; a several-second per-pixel loop on the UI thread would
   freeze the page.
4. **Clamp every computed bucket index** to `count - 1` - a pure-white pixel
   maps exactly onto `count` (`HW-18-0080`).
5. **Write channels back into their own bytes.** RGBA_8888 is byte 0 = R,
   byte 2 = B; the oil kernel's accumulator returns `[B, G, R, A]`
   (`HW-18-0081`).
6. **Attach a `.catch` to `taskpool.execute`** that clears the loading flag and
   tells the user, otherwise a kernel throw leaves a full-screen overlay
   forever (`HW-18-0082`).
7. **Rebuild the `PixelMap`** from the returned buffer with
   `image.createPixelMapSync` and the same `pixelFormat` and size.
8. **Save through `showAssetsCreationDialog`** - pack to JPEG, write a temp
   file in `cacheDir`, get its `fileuri`, hand it to the dialog, and copy into
   the returned destination uri.
9. **Release the `ImagePacker`** after packing (`HW-18-0036`), and take the
   `UIContext` from the page rather than constructing one (`HW-18-0083`).

## Verified snippets

All snippets are from `ImageFilterProcessing.zip`. Corrected forms are marked.

**The oil-painting bucket — `entry/src/main/ets/utils/OilPaintingFilterUtil.ets`** (corrected, see `HW-18-0080`, `HW-18-0081`)

```typescript
export class ColorTank {
  private count: number = 0;
  private colorList: List<List<number>> = new List<List<number>>();

  static rgb2gray(rgb: number) {
    let b = (rgb >> 16) & 0x000000ff;
    let g = (rgb >> 8) & 0x000000ff;
    let r = (rgb >> 0) & 0x000000ff;
    let gray = (r * 0.299 + g * 0.587 + b * 0.114);
    return gray;
  }

  ColorTank(count: number) {
    this.count = count;
    for (let i = 0; i < this.count; i++) {
      this.colorList.add(new List<number>());
    }
  }

  addToTank(rgb: number) {
    let gray = ColorTank.rgb2gray(rgb);
    // FIX: shipped `Math.floor((gray / 255) * this.count)` yields `count` when gray === 255
    let index = Math.min(this.count - 1, Math.floor((gray / 255) * this.count));
    this.colorList.get(index).add(rgb);
  }
}
```

**The buckets are `0 .. count - 1`, but the mapping can produce `count`.**
`gray / 255` reaches exactly `1.0` on a pure white pixel, so
`Math.floor(1.0 * count)` is `count`, and `colorList.get(count)` is out of
range - `.add` on the result throws. White is not an edge case in photography:
a highlight, a sky, a white background or a blown-out pixel all hit it, so the
filter throws on most real images. Combined with the missing `.catch`
(`HW-18-0082`) the app then hangs behind the spinner. `Math.min` against
`count - 1` is the whole fix; the alternative, `Math.floor(gray / 256 * count)`,
also works.

**The channel write-back — same file** (corrected, see `HW-18-0081`)

```typescript
  getColor() {
    // ... find maxIndex = the bucket with the most pixels ...
    let maxRGBList = this.colorList.get(maxIndex);
    let allColorR = 0;
    let allColorG = 0;
    let allColorB = 0;
    let allColorA = 255;
    for (let i = 0; i < maxRGBList.length; i++) {
      let value: number = maxRGBList.get(i);
      allColorB += ((value >> 16) & 0x000000ff);
      allColorG += ((value >> 8) & 0x000000ff);
      allColorR += ((value >> 0) & 0x000000ff);
    }
    allColorR = allColorR / maxRGBList.length;
    allColorG = allColorG / maxRGBList.length;
    allColorB = allColorB / maxRGBList.length;
    return [allColorB, allColorG, allColorR, allColorA];   // note the order: B, G, R, A
  }

// in pixelMapOilPaintingFilter, per output pixel:
let pixelColor = ctank.getColor();
let indexTemp = y * width + x;
byteBufferTemp[indexTemp * 4] = pixelColor[2];       // FIX: R into byte 0 (shipped: pixelColor[0])
byteBufferTemp[indexTemp * 4 + 1] = pixelColor[1];   // G
byteBufferTemp[indexTemp * 4 + 2] = pixelColor[0];   // FIX: B into byte 2 (shipped: pixelColor[2])
byteBufferTemp[indexTemp * 4 + 3] = pixelColor[3];   // A
```

**Two views of the same memory, read in opposite orders.** The input is read as
a `Uint32Array`, so on a little-endian device the low byte of each word is
RGBA byte 0, which is **red** - and `rgb2gray` gets this right, taking `r` from
bits 0-7. `getColor` then accumulates the same bits into `allColorR` correctly,
but returns them in the order `[B, G, R, A]`. The output is written through a
`Uint8Array`, where index 0 is again red. The shipped write-back puts
`pixelColor[0]` - blue - into the red byte. The result is visibly wrong on
every image: oranges come out blue, blue skies come out orange. Swapping the
two indices at the write-back is the minimal fix; renaming `getColor`'s return
to `[R, G, B, A]` is the cleaner one.

The rest of the algorithm is sound and worth understanding: for each pixel it
walks a `(2 * radius)`-square neighbourhood, drops every neighbour into one of
`tank` grey-level buckets, finds the fullest bucket, and averages its colours.
That "most common tone in the brush footprint" is what produces the flat,
palette-knife regions. `MainPage` passes `radius = 5` and `tank = 6`.

**Dispatching to the taskpool — `entry/src/main/ets/pages/MainPage.ets`** (corrected, see `HW-18-0082`)

```typescript
@Concurrent
function imageFilterProcessing(imgInfo: ImageInfo, filterType: ImageFilterType) {
  let buffer: ArrayBuffer = new ArrayBuffer(4);
  if (filterType === ImageFilterType.OILPAINTING_FILTER) {
    buffer =
      pixelMapOilPaintingFilter(imgInfo.readBuffer, imgInfo.width, imgInfo.height, imgInfo.radius, imgInfo.tank);
  }
  if (filterType === ImageFilterType.PENCIL_SKETCH_FILTER) {
    buffer = imageProcessByPencilSketch(imgInfo.readBuffer, imgInfo.width, imgInfo.height, imgInfo.radius);
  }
  let imageInfo = new ImageInfo();
  imageInfo.readBuffer = buffer;
  imageInfo.width = imgInfo.width;
  imageInfo.height = imgInfo.height;
  imageInfo.radius = imgInfo.radius;
  return imageInfo;
}

// the filter strip handler:
.onClick(() => {
  this.currentIndex = index;
  this.isPixelMapChange = false;
  this.isProcessEnd = false;
  if (this.currentIndex === 0) {
    this.pixelMapTemp = this.pixelMap;            // original: no task, flags reset immediately
    this.isPixelMapChange = true;
    this.isProcessEnd = true;
  } else {
    let pixelMap: PixelMap = this.pixelMap as image.PixelMap;
    let imageInfoTemp: ImageInfo = getPixelMapInfo(pixelMap);
    let filterType: ImageFilterType = ImageFilterType.PENCIL_SKETCH_FILTER;
    if (this.currentIndex === 1) {
      imageInfoTemp.radius = 5;                   // brush size
      imageInfoTemp.tank = 6;                     // colour levels
      filterType = ImageFilterType.OILPAINTING_FILTER;
    }
    if (this.currentIndex === 2) {
      imageInfoTemp.radius = 10;                  // pencil line thickness
      filterType = ImageFilterType.PENCIL_SKETCH_FILTER;
    }
    taskpool.execute(imageFilterProcessing, imageInfoTemp, filterType)
      .then((res: Object) => {
        let imgInfo = res as ImageInfo;
        this.pixelMapTemp = imageEncode(imgInfo);
        this.isPixelMapChange = true;
        this.isProcessEnd = true;
      })
      .catch((err: Error) => {                    // FIX: absent in the sample
        this.isProcessEnd = true;                 //      the overlay never lifts on failure
        this.getUIContext().getPromptAction().showToast({ message: $r('app.string.filter_failed') });
      });
  }
})
```

**`isProcessEnd` is the whole progress model, and it has one owner.** It gates
the `LoadingProgress` overlay's `visibility` and the save icon's `.enabled()`.
It is cleared before the dispatch and set again in `.then` - so with no
`.catch`, a rejected task leaves a full-screen, tap-blocking overlay up until
the app is restarted, plus an unhandled rejection. Given the bucket crash above,
rejection is not hypothetical; it is the common case on an image with white in
it.

Note that the `@Concurrent` function takes the enum and the DTO as *arguments*
rather than closing over anything: a concurrent function runs in a different
context and can capture nothing. That constraint is why `radius` and `tank` are
fields on `ImageInfo` rather than constants read inside the kernel.

**Saving without a permission — `entry/src/main/ets/utils/ImageUtil.ets`** (corrected, see `HW-18-0083`, `HW-18-0036`)

```typescript
export async function savePixelMapToAlbum(pixelMap: image.PixelMap, context: Context, uiContext: UIContext) {
  const dateStr = (new Date().getTime()).toString();
  let tempPath = context.cacheDir + `/${'Temp' + dateStr}.png`;    // app sandbox, not the gallery
  let file: fileIo.File | null = null;
  try {
    file = fileIo.openSync(tempPath, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
    const imageBuffer = await packingPixelMap2Jpg(pixelMap as image.PixelMap);
    fileIo.writeSync(file.fd, imageBuffer);
  } finally {
    if (file !== null) {
      fileIo.closeSync(file.fd);
    }
  }

  let resFile: fileIo.File | null = null;
  let desFile: fileIo.File | null = null;
  try {
    let realUri = fileuri.getUriFromPath(tempPath);
    let phAccessHelper = photoAccessHelper.getPhotoAccessHelper(context);
    let photoCreationConfigs: Array<photoAccessHelper.PhotoCreationConfig> = [
      { fileNameExtension: 'png', photoType: photoAccessHelper.PhotoType.IMAGE,
        subtype: photoAccessHelper.PhotoSubtype.DEFAULT }
    ];
    let desFileUris: Array<string> = await phAccessHelper.showAssetsCreationDialog([realUri], photoCreationConfigs);
    resFile = fileIo.openSync(realUri, fileIo.OpenMode.READ_ONLY);
    desFile = fileIo.openSync(desFileUris[0], fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
    fileIo.copyFileSync(resFile.fd, desFile.fd);
    uiContext.getPromptAction().showToast({ message: '保存成功', duration: 2000 });  // FIX: was `new UIContext()`
  } catch (err) {
    console.error(`showAssetsCreationDialog failed, errCode is: ${err.code} errMsg is: ${err.message}`);
  } finally {
    if (resFile !== null) {
      fileIo.closeSync(resFile.fd);
    }
    if (desFile !== null) {
      fileIo.closeSync(desFile.fd);
    }
  }
}
```

**`showAssetsCreationDialog` is the reason this sample declares no
permissions.** The app writes only into its own `cacheDir`, hands the system a
sandbox uri, and the user confirms the gallery entry in a system dialog; the
returned uri is a handle the app may write once. No WRITE_IMAGEVIDEO, no ACL in
the signing profile, no runtime prompt of your own. The cost is that it is
interactive, so it suits an explicit save action and not a batch export.

Two things around it need correcting. `new UIContext()` builds an object bound
to no window, so its `showToast` fails and the surrounding `catch` swallows the
error - the user gets no confirmation at all (`HW-18-0083`); pass the page's
`this.getUIContext()` in. And `packingPixelMap2Jpg` creates an `ImagePacker`
per save and never calls `release()`, leaking a native packer every time
(`HW-18-0036`, eight samples in this industry) - wrap the `packToData` in a
`try/finally` with `imagePackerApi.release()`.

Note also `fileNameExtension: 'png'` against `format: 'image/jpeg'` in the
packing options: the bytes are JPEG, the gallery entry is named `.png`.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions` block at all, which is
the point of the `showAssetsCreationDialog` flow. `deviceTypes` is `["phone"]`.

The source image is `rawfile/fruitImg.png`, read through
`resourceManager.getRawFdSync`. The in-source comment says so explicitly:
`fruitImg.png为测试图片，开发者需要在rawfile目录下替换为实际图片` (this is a test
image, replace it with a real one in the rawfile directory). There is no picker
in this sample - wiring one to `photoAccessHelper` is the first change a real
app makes.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The oil-painting kernel is O(width x height x radius^2) with a `List`
  allocation per pixel: at `radius = 5` that is 100 neighbour reads and six
  fresh `List` objects for every pixel. It is seconds-scale on a phone photo
  and is the reason the spinner exists. Downscale before filtering if the
  source is large.
- The pencil kernel is much cheaper - the Gaussian blur is separable into a
  horizontal and a vertical pass, `O(width x height x radius)` - and its
  `createGaussianKernel1D` normalises the kernel, so no scaling is needed after
  the convolution.
- `applyPaperTexture` indexes a fixed 7x5 table with `x % 7` and `y % 5`, so the
  texture tiles visibly at that period on a flat region.
- `desiredPixelFormat: 3` (RGBA_8888) is assumed by both kernels; a different
  decode format would break the byte arithmetic silently.
- The cut / filter / adjust / graffiti bar at the bottom is decorative - only
  the filter strip is wired.
- Temp files accumulate in `cacheDir`: nothing deletes `Temp<timestamp>.png`
  after the copy.

## Pitfalls

- **`HW-18-0080`** (B/medium, confirmed): `addToTank` computes
  `Math.floor((gray / 255) * count)`, which equals `count` for a pure white
  pixel while only buckets `0 .. count - 1` exist, so `colorList.get(count).add`
  throws on any image with a highlight or a white background. Fix:
  `Math.min(count - 1, ...)`.
- **`HW-18-0081`** (B/medium, confirmed): `getColor` returns `[B, G, R, A]` and
  the write-back puts index 0 into RGBA byte 0, so red and blue are swapped in
  every oil-painting result. Fix: write `pixelColor[2]` to byte 0 and
  `pixelColor[0]` to byte 2.
- **`HW-18-0082`** (B/medium, confirmed): `taskpool.execute` has no `.catch`, so
  a kernel failure - realistic given `HW-18-0080` - leaves `isProcessEnd` false
  and the full-screen `LoadingProgress` overlay blocking the UI permanently.
  Fix: `.catch` that resets the flag and toasts.
- **`HW-18-0083`** (B/low, probable): the success toast is raised from
  `new UIContext()`, an instance bound to no window; `showToast` fails and the
  surrounding `catch` hides it, so the save silently gives no feedback. Fix:
  pass the page's `getUIContext()` into `savePixelMapToAlbum`.
- **`HW-18-0036`** (B/low, confirmed): `packingPixelMap2Jpg` creates an
  `ImagePacker` on every save and never releases it; native memory accumulates
  for the process lifetime. Eight samples in this industry share it
  (`CompressImages` is the one that gets it right). Fix:
  `imagePackerApi.release()` in a `finally`.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-image-imagesource.md` - `createImageSource`, `createPixelMapSync`, `DecodingOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagesource
- `documentation/harmonyos-references/04_media/arkts-apis-image-PixelMap.md` - `readPixelsToBufferSync`, `getImageInfoSync`, pixel formats
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-pixelmap
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `showAssetsCreationDialog` and `PhotoCreationConfig`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-references/02_application-framework/js-apis-taskpool.md` - `taskpool.execute` and the `@Concurrent` restrictions
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-taskpool
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `openSync`, `writeSync`, `copyFileSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/02_application-framework/js-apis-resource-manager.md` - `getRawFdSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-resource-manager
- `documentation/harmonyos-references/02_application-framework/js-apis-arkui-UIContext.md` - why a `UIContext` must come from a bound window
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-arkui-uicontext
- `documentation/harmonyos-guides/05_media/image-pixelmap-operation.md` - the buffer round-trip pattern
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-pixelmap-operation
- `documentation/harmonyos-guides/03_application-framework/taskpool-introduction.md` - when to move work off the UI thread
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/taskpool-introduction
- `PHOTO-08` - the `ImagePacker` leak (`HW-18-0036`) across the industry
