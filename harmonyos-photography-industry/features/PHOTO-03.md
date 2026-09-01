---
id: PHOTO-03
title: Gallery photo filters - effectKit color matrix over a PixelMap, saved through SaveButton
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/03_image_filter.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_filter-0000002284048625
sample: huawei_industry_tree/18_photography/downloads/ImageFilter.zip
kits: ["@kit.ImageKit", "@kit.ArkGraphics2D", "@kit.MediaLibraryKit", "@kit.CoreFileKit", "@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: ["effectKit.createEffect", setColorMatrix, grayscale, brightness, getEffectPixelMap, "image.createImageSource", createPixelMap, "image.createImagePacker", packToData, "photoAccessHelper.PhotoViewPicker", PhotoSelectOptions, "photoAccessHelper.getPhotoAccessHelper", createAsset, SaveButton, SaveButtonOnClickResult, "fileIo.openSync", "fileIo.closeSync", "window.getWindowAvoidArea"]
permissions: ["ohos.permission.WRITE_IMAGEVIDEO"]
min_api: 20
modules: [entry]
findings: [HW-18-0001, HW-18-0024, HW-18-0025, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when the app has to **apply a colour filter to a photo and hand
the result back to the gallery** - the last two steps of any photo editor:
turn a picked image into a `PixelMap`, run a colour transform over it, encode
and write it as a new album asset.

The pattern is: `PhotoViewPicker` in, `effectKit.createEffect(pixelMap)` in the
middle, `SaveButton` + `createAsset` out. None of the three needs a runtime
permission request. The picker returns URIs the app is temporarily allowed to
read; `SaveButton` grants a ten-second window in which `createAsset` may write
one file. That is the whole security story, and it is why this shape is worth
copying even when the filter itself is something else - a LUT, a blur, a
taskpool pixel kernel.

The transferable part is the **two-`PixelMap` state machine**. The sample keeps
the source image and the filtered image side by side and flips between them
with one boolean, so "original" is free, filters never stack accidentally, and
re-picking an image resets to a known state. Any non-destructive editor wants
that shape.

## Feature checklist

- The page opens on a bundled `rawfile/image.jpg` already decoded into a
  `PixelMap` - the editor is never empty.
- A 滤镜 (filter) button expands a strip of four round filter chips: 原图
  (original), 高亮 (bright), 粉色 (pink), 灰度 (grayscale).
- Tapping a chip applies the filter to the source image and swaps the preview;
  the chosen chip switches to its highlighted icon.
- 原图 returns to the unfiltered image without re-decoding it.
- A `+` button opens the system gallery picker; the picked photo replaces the
  source and the filter selection resets to 原图.
- Cancelling the picker leaves the current image in place (see `HW-18-0024`).
- `SaveButton` writes the filtered image into the gallery as a PNG and toasts
  保存成功.
- Tapping save with no filter applied toasts a "nothing to save" message
  instead of writing a duplicate.
- 取消 (cancel) collapses the filter strip back to the single button.

## Architecture

One `entry` module, 260 lines of page and two small utility files. No model
layer beyond an enum and an icon list.

```
entry/src/main/ets
├── constants/CommonConstants.ets     every literal size, margin and percentage
├── entryability/EntryAbility.ets     dark colour mode, full-screen, avoid areas -> AppStorage
├── entrybackupability/
├── pages/MainPage.ets                @Entry: preview + filter strip + footer, all state
├── utils
│   ├── FilterUtil.ets                the three effectKit filters, 3 pure functions
│   └── ImageUtil.ets                 rawfile -> PixelMap, gallery -> PixelMap, uri -> buffer
└── viewModel
    ├── IconListViewModel.ets         IconStatus + the four chips (normal/chosen icon + label)
    └── OptionViewModel.ts            enum FilterType - note the .ts extension
```

The documented tree lists `viewModel/OptionViewModel.ets`; the file in the zip
is `OptionViewModel.ts` (`HW-18-0001`). It is only an extension, but it is one
of five trees in this industry that were not regenerated after the samples
changed, and a reader navigating by the document will not find the file.

**The design decision worth copying** is that `FilterType` is an enum whose
numeric values are also the chip indices. `filterIconList` is declared in the
same order as the enum, so the `ForEach` index *is* the filter type and the
click handler is one line:

```typescript
this.currentFilterIndex = index;
await this.filterImage(index);          // index used directly as FilterType
```

There is no lookup table, no `id` field on the chip, and no mapping to keep in
sync. The cost is that reordering the chips silently reorders the filters, so
the two declarations have to stay adjacent - which they are, one file apart.
For a fixed, small set of options this is the right trade; for a filter list
loaded from a server it would not be.

## Implementation steps

1. **Seed the editor from a bundled rawfile** in `aboutToAppear` so the page
   has something to filter before the user picks anything.
2. **Decode through a cache file**: `getRawFileContent` returns a buffer;
   write it to `cacheDir`, open it, and build the `ImageSource` from the fd.
   **Close the fd only after `createPixelMap` resolves** (`HW-18-0025`).
3. **Decode with `editable: true`** - `effectKit` needs a writable bitmap.
4. **Keep two pixel maps and one boolean.** `pixelMap` is the source,
   `pixelMapChanged` the filtered result, `isPixelMapChange` selects which one
   the `Image` renders. Always filter from `pixelMap`, never from the previous
   result, or the filters compound.
5. **Guard the picker's cancel path**: `photoSelectResult.photoUris` is empty
   when the user backs out, and `uris[0]` is then `undefined`
   (`HW-18-0024`).
6. **Apply the filter with `effectKit.createEffect(pixelMap).<op>().getEffectPixelMap()`**
   and `await` the result - `getEffectPixelMap` is asynchronous.
7. **Save inside the `SaveButton` click handler**, checking
   `result === SaveButtonOnClickResult.SUCCESS` first, and call `createAsset`
   within ten seconds of the click - the temporary grant expires.
8. **Pack, then write through the returned URI.** `createAsset` gives a URI
   whose fd may be written without a time limit; only `createAsset` itself is
   time-boxed.

## Verified snippets

All snippets are from `ImageFilter.zip`. Corrected forms are marked.

**The three filters — `entry/src/main/ets/utils/FilterUtil.ets`** (as shipped)

```typescript
import { effectKit } from '@kit.ArkGraphics2D';

export async function pinkColorFilter(pixelMap: PixelMap) {
  const pinkColorMatrix: Array<number> = [
    1, 1, 0, 0, 0,
    0, 1, 0, 0, 0,
    0, 0, 1, 0, 0,
    0, 0, 0, 1, 0
  ];
  const pixelMapFiltered = await effectKit.createEffect(pixelMap).setColorMatrix(pinkColorMatrix).getEffectPixelMap();
  return pixelMapFiltered;
}

export async function grayImageFilter(pixelMap: PixelMap) {
  const pixelMapFiltered = await effectKit.createEffect(pixelMap).grayscale().getEffectPixelMap();
  return pixelMapFiltered;
}

export async function brightImageFilter(pixelMap: PixelMap) {
  let bright = 0.5;
  const pixelMapFiltered = await effectKit.createEffect(pixelMap).brightness(bright).getEffectPixelMap();
  return pixelMapFiltered;
}
```

**`createEffect(...).<op>().getEffectPixelMap()` is a builder, not a mutator.**
Each `<op>` appends a stage to a chain and returns the same `Filter`, so
`grayscale().brightness(0.5)` would be a two-stage pipeline; `getEffectPixelMap`
is what actually runs it and produces a **new** `PixelMap`. The source is left
untouched, which is exactly what makes the two-pixel-map state machine work.

The matrix is the general escape hatch: a 4x5 row-major matrix mapping RGBA to
RGBA plus a constant column. The "pink" one is the identity with a single `1`
added at row 0, column 1 - red picks up the full green channel, so mid-greens
push red up and the image warms towards magenta. `grayscale()` and
`brightness()` are named shortcuts for matrices you would otherwise write out.

**Decoding a rawfile — `entry/src/main/ets/utils/ImageUtil.ets`** (corrected, see `HW-18-0025`)

```typescript
export async function getPixelMapFromRaw(context: Context) {
  const resourceMgr = context.resourceManager;
  let imageBuffer = await resourceMgr.getRawFileContent('image.jpg');
  let filePath = context.cacheDir + '/' + 'test.png';
  let file = fileIo.openSync(filePath, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
  fileIo.writeSync(file.fd, imageBuffer.buffer);
  const imageSourceApi = image.createImageSource(file.fd);
  try {
    const pixelMap = await imageSourceApi.createPixelMap({
      editable: true                       // effectKit needs a writable bitmap
    });
    return pixelMap;
  } finally {
    imageSourceApi.release();              // FIX: sample never releases the source
    fileIo.closeSync(file.fd);             // FIX: sample closes here, BEFORE the decode above
  }
}
```

**The fd must outlive the decode.** `createImageSource(fd)` does not read the
file eagerly; it keeps the descriptor and pulls bytes when `createPixelMap`
runs. The shipped code calls `fileIo.closeSync(file.fd)` on the line straight
after `createImageSource` and before the `await`, so the decode races a closed
descriptor. It usually survives on a fast local cache file, which is why the
same three-line sequence appears in `PictureSticker` and `ImageDenoising` too
(`HW-18-0025`), but the contract does not allow it.

Note the round trip through `cacheDir`. `createImageSource` also accepts an
`ArrayBuffer` directly - the gallery path below uses that form - so the file
write here is avoidable. It is worth keeping only if the same image is decoded
repeatedly and the cache copy is reused, which this sample does not do.

**The picker, with the cancel path closed — same file** (corrected, see `HW-18-0024`)

```typescript
export async function getPixelMapFromGallery() {
  const photoSelectOptions: photoAccessHelper.PhotoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
  photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
  photoSelectOptions.maxSelectNumber = 1;
  let uris: Array<string> = [];
  let pixelMap: image.PixelMap | undefined = undefined;
  const photoViewPicker = new photoAccessHelper.PhotoViewPicker();
  await photoViewPicker.select(photoSelectOptions)
    .then(async (photoSelectResult: photoAccessHelper.PhotoSelectResult) => {
      uris = photoSelectResult.photoUris;
      if (uris.length === 0) {                 // FIX: absent in the sample
        return;                                //      cancel resolves with an empty array
      }
      let imageBuffer = getBufferByUri(uris[0]);
      let imageSource: image.ImageSource = image.createImageSource(imageBuffer);
      if (!imageSource) {
        return;
      }
      pixelMap = await imageSource.createPixelMap({ editable: true });
      imageSource.release();
    })
    .catch(() => {                             // FIX: no catch in the sample
      pixelMap = undefined;
    });
  return pixelMap;
}
```

**Cancelling the picker is a success, not an error.** `select` resolves
normally with `photoUris: []`, so the shipped code reaches
`getBufferByUri(undefined)` and `fileIo.openSync` throws inside a `.then` that
nobody catches - an unhandled rejection on the most ordinary user action there
is. Six samples in this industry share the defect; `ImageRotateAndFlip` carries
the empty-array guard and shows the intended shape.

`maxSelectNumber = 1` plus `IMAGE_TYPE` is what keeps `uris[0]` meaningful.
Raise the count and the caller has to loop; the guard becomes `for (const uri
of uris)` and the single-`PixelMap` return type has to change with it.

**The save path — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
SaveButton({ text: SaveDescription.SAVE })
  .onClick(async (event: ClickEvent, result: SaveButtonOnClickResult) => {
    if (this.isPixelMapChange) {
      if (result === SaveButtonOnClickResult.SUCCESS) {
        const imagePackerApi: image.ImagePacker = image.createImagePacker();
        let packOpts: image.PackingOption = { format: 'image/png', quality: 98 };
        this.arrayBuffer =
          await imagePackerApi.packToData(this.isPixelMapChange ? this.pixelMapChanged : this.pixelMap, packOpts);
        let helper = photoAccessHelper.getPhotoAccessHelper(this.getUIContext().getHostContext()!);
        // onClick触发后10秒内通过createAsset接口创建图片文件，10秒后createAsset权限收回。
        let uri = await helper.createAsset(photoAccessHelper.PhotoType.IMAGE, 'png');
        // 使用uri打开文件，可以持续写入内容，写入过程不受时间限制
        let file = await fileIo.open(uri, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
        await fileIo.write(file.fd, this.arrayBuffer);
        await fileIo.close(file.fd);
        this.getUIContext().getPromptAction().showToast({
          message: $r('app.string.save_successfully'),
          duration: CommonConstants.DURATION
        });
      } else {
        this.getUIContext().getPromptAction().showToast({ message: $r('app.string.save_failed') });
      }
    } else {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.save_disable') });
    }
  });
```

**The two comments are the load-bearing part of this handler.** `SaveButton` is
a security component: the click grants a temporary write capability that
`createAsset` consumes, and it expires ten seconds after the click. So
`createAsset` must be reached quickly - and `packToData` runs *before* it here,
which for a large image eats into that budget. Inverting the order (create the
asset first, then pack and write) is the safer arrangement for full-resolution
photos. Once the URI exists, the `fileIo.open`/`write`/`close` sequence has no
time limit at all, as the second comment says.

`result` must be checked: the framework passes `SaveButtonOnClickResult` and a
non-`SUCCESS` value means the grant was refused, in which case `createAsset`
would throw. The outer `isPixelMapChange` test is a product decision - saving
an unfiltered image would just duplicate the original.

The packer created on line one is never released, the industry-wide
`ImagePacker` leak catalogued as `HW-18-0036` for eight sibling samples;
`CompressImages` shows the correct `release()` in a `finally`.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": 'ohos.permission.WRITE_IMAGEVIDEO',
    "reason": '$string:reason',
    "usedScene": { "abilities": ["EntryAbility"], "when": 'always' }
  }
]
```

**This declaration is unnecessary and harmful.** `WRITE_IMAGEVIDEO` is a
restricted (ACL) permission that an ordinary app cannot ship with, and nothing
in this sample needs it: the read side goes through `PhotoViewPicker`, the
write side through `SaveButton` + `createAsset`, and both were designed
precisely so that no album permission is required. The sample never calls
`requestPermissionsFromUser` for it either. This is the same shared
`module.json5` template defect catalogued as `HW-18-0004` against nine other
photography samples - `ImageFilter` is a tenth instance. Delete the entry.

The ability also fixes the colour mode:
`setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_DARK)` in `onCreate`,
which is defensible for a photo editor (a neutral dark surround is the standard
for judging colour) but means the app ignores the system theme. The avoid areas
are read once in `onWindowStageCreate` into `AppStorage` as `topAvoid` /
`bottomAvoid` and applied as page padding - with no `avoidAreaChange`
subscription, so a rotation or a resized 2in1 window leaves stale padding.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- `deviceTypes` is `phone`, `tablet`, `2in1`, but the layout is percentage-based
  around a single portrait column (77.5% preview, 30% controls - note these do
  not sum to 100%) and was not designed for a wide window.
- `getEffectPixelMap` allocates a full new bitmap per invocation. Tapping
  through the four chips on a 12 MP image allocates four of them; nothing is
  released, so the page holds every filtered result the user has tried.
- The output is always PNG at quality 98 regardless of the source format, so a
  JPEG in becomes a considerably larger PNG out.
- Filters apply to the whole image only - there is no mask, no intensity
  slider, and no undo beyond re-selecting 原图.

## Pitfalls

- **`HW-18-0001`** (E/low, confirmed): the documented 工程目录 lists
  `OptionViewModel.ets` but the zip ships `OptionViewModel.ts`. One of five
  photography trees that do not match their zips. Fix: regenerate the tree
  blocks from the current zips.
- **`HW-18-0024`** (B/low, confirmed): the photo-picker cancel path is
  unhandled - `photoUris` is empty, `uris[0]` is `undefined`, and
  `fileIo.openSync(undefined)` throws inside a `.then` with no `.catch`.
  Systematic across six samples. Fix: guard on `photoUris.length === 0` and add
  a `.catch` to the chain.
- **`HW-18-0025`** (B/low, probable): `getPixelMapFromRaw` closes the fd
  immediately after `createImageSource(file.fd)` and before the awaited
  `createPixelMap`, so the decode may run against a closed descriptor and leave
  the startup image blank. Systematic across three samples. Fix: move
  `closeSync` into a `finally` after the decode.
- **Restricted permission declared without need** (related to `HW-18-0004`):
  `WRITE_IMAGEVIDEO` is in `module.json5` although the sample uses only the
  permission-free `SaveButton` and picker flows. Fix: delete the entry.
- **`ImagePacker` never released** (same defect as `HW-18-0036`): every save
  leaks one native packer. Fix: `imagePackerApi.release()` in a `finally`.
- **`ForEach` keyed on `JSON.stringify(item)`** where `item` holds two
  `Resource` objects - a stable but needlessly expensive key. The index would
  do for a fixed four-element list.

## References

- `documentation/harmonyos-references/05_graphics/js-apis-effectkit.md` - `createEffect`, `setColorMatrix`, `grayscale`, `brightness`, `getEffectPixelMap`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-effectkit
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` - `SaveButton`, `SaveDescription`, `SaveButtonOnClickResult` and the temporary grant
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `PhotoViewPicker.select`, `PhotoSelectOptions`, `PhotoSelectResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `createAsset` and its ten-second window
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `openSync`, `closeSync`, `write`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-guides/02_media/image-decoding.md` - `createImageSource`, `createPixelMap`, and releasing the source
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/image-decoding
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - why `WRITE_IMAGEVIDEO` should not be declared here
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `PHOTO-28` - the same filter slot done by hand in a taskpool, with the pixel-format traps that avoids
