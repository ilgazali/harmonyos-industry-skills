---
id: PHOTO-12
title: GIF from a picture sequence - packToFileFromPixelmapSequence behind an API-version gate
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/12_gif_generator.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/gif_generator-0000002330170016
sample: huawei_industry_tree/18_photography/downloads/GIFGenerator.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.MediaLibraryKit"]
apis: [common, deviceInfo, fileIo, fileUri, fs, hilog, image, photoAccessHelper, window]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0005, HW-18-0024, HW-18-0036, HW-18-0040, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card for two independent things that happen to ship in one sample.

The first is **animated-GIF encoding from a list of pixel maps**:
`ImagePacker.packToFileFromPixelmapSequence` takes an array of `PixelMap` plus
a `PackingOptionsForSequence` describing frame count, per-frame delays,
disposal behaviour and loop count, and writes a GIF. It is the only supported
route to a multi-frame image on HarmonyOS, and it works for anything you can
produce as pixel maps - burst photos, rendered frames, a scanned document
flipbook, a chart animation.

The second is **shipping a feature that only exists above a certain API
level**. `packToFileFromPixelmapSequence` arrived in API 18, but the app has to
install and run on older devices. The sample reads `deviceInfo.sdkApiVersion`
into state and lets the layout decide, per entry, whether a tile is rendered at
all. That pattern - runtime capability gate driving the layout rather than a
try/catch around the call - is worth copying whenever a Kit API's availability
version is below your `compatibleSdkVersion` floor.

**Read `HW-18-0040` before adopting the caching approach.** The sample wipes
and recreates the app cache directory on every page appearance, with two
un-awaited async calls racing each other.

## Feature checklist

- A home page with a banner image, a 拍摄照片 (take photo) button and a
  four-tile module row.
- On devices below API 18 the fourth tile (GIF composition) is not rendered at
  all; the other three are.
- On API 18 and above all four tiles render, and the fourth is tinted blue to
  mark it as the live one.
- Tapping the GIF tile opens the picker, images only, up to nine.
- Each picked image is decoded to an RGBA_8888 editable pixel map.
- The sequence is encoded into a GIF in the app cache directory, named with a
  timestamp.
- The app navigates to a display page showing the animated result.
- A `SaveButton` on that page copies the GIF into the album and confirms with
  an alert dialog.

## Architecture

One `entry` module. The GIF logic lives in the module-row view, not in a
utility - which is the sample's one structural weakness.

```
entry/src/main/ets
├── common/Constants.ets            API_18, INDEX_FORTH, MAX_SELECT_NUMBER, GIF frame params
├── entryability/EntryAbility.ets   full screen, avoid areas -> AppStorage
├── entrybackupability/
├── model/ModuleItem.ets            ModuleItem + the four-entry moduleData array
├── pages
│   ├── MainPage.ets                @Entry: Navigation host, cache-dir reset
│   └── Show.ets                    NavDestination: GIF preview + SaveButton
├── utils/Logger.ets                hilog wrapper
└── views
    ├── Banner.ets                  background image with a blend-mode fade
    ├── Module.ets                  the four tiles AND the whole GIF pipeline
    └── Shoot.ets                   a static button, no handler
```

The documented tree matches the zip.

**The design decision worth copying** is the version gate as a *layout*
condition, not an error handler:

```typescript
@State sdkApiVersionInfo: number = deviceInfo.sdkApiVersion;

ForEach(this.entryArr, (item: ModuleItem, index: number) => {
  if (this.sdkApiVersionInfo < Constants.API_18 && index !== Constants.INDEX_FORTH) {
    // three tiles, no click handler
  } else if (this.sdkApiVersionInfo >= Constants.API_18) {
    // four tiles; the fourth is blue and clickable
  }
});
```

Read the two conditions together: below API 18 the tile at `INDEX_FORTH` (3)
matches neither branch, so it is never created. The user of an old device does
not see a disabled control or a failing one - the feature is simply absent, and
the remaining three tiles reflow across the `Flex`. `deviceInfo.sdkApiVersion`
is read once into `@State` rather than queried per render.

**The decision worth avoiding** is that `Module.ets` is a presentational
component containing the entire picker → decode → encode → navigate pipeline
in one 48-line async method. Everything in `GifGenerator()` below the picker
call belongs in a utility; as shipped, the GIF feature cannot be reused, tested
or invoked from anywhere else, and the view re-renders on `@State uri` changes
that have nothing to do with its layout. Compare `PHOTO-09`'s
`PictureUtils` singleton, which is the same pipeline factored correctly.

## Implementation steps

1. **Read `deviceInfo.sdkApiVersion` once into `@State`** and branch the layout
   on it. Do not call the API and catch the failure.
2. **Reset the working directory deterministically.** If you clear a cache
   directory on entry, `await` the removal before the creation - the shipped
   `fs.rmdir(path); fs.mkdir(path);` is two un-awaited promises racing
   (`HW-18-0040`). Better still, delete only your own `result_*.gif` files
   instead of the whole `cacheDir`, which other subsystems also use.
3. **Pick with `maxSelectNumber = 9`** and images only.
4. **Guard the cancel path.** An empty `photoUris` currently still opens a
   file, writes a zero-byte GIF and navigates to a blank preview
   (`HW-18-0024`).
5. **Decode each URI to a pixel map** with
   `desiredPixelFormat: RGBA_8888` and `editable: true`, and close each source
   descriptor as you go.
6. **Release each `ImageSource`** after its pixel map is produced
   (`HW-18-0005`), and release the pixel maps once the pack completes.
7. **Open the output file** in `cacheDir` under a timestamped name, so a second
   run cannot collide with a GIF the preview page is still reading.
8. **Build `PackingOptionsForSequence`** with `frameCount` equal to the number
   of pixel maps, a `delayTimeList` in units of 10 ms, `disposalTypes`, and
   `loopCount: 0` for an infinite loop. Give `delayTimeList` and `disposalTypes`
   one entry per frame - the sample passes single-element arrays for a
   nine-frame GIF.
9. **`await packToFileFromPixelmapSequence(pixelMapList, file.fd, ops)`**,
   release the packer in `finally` (`HW-18-0036`), and close the file there
   too.
10. **Navigate with the file URI**, not the path:
    `fileUri.getUriFromPath(path)` then `pushPathByName('show', this.uri)`.
11. **Save with `SaveButton` + `createImageAssetRequest` + `applyChanges`,**
    guarded on `SaveButtonOnClickResult.SUCCESS`.

## Verified snippets

All snippets are from `GIFGenerator.zip`. Corrected forms are marked.

**The cache reset — `entry/src/main/ets/pages/MainPage.ets`**
(corrected, see `HW-18-0040`)

```typescript
import fs from '@ohos.file.fs';

async aboutToAppear(): Promise<void> {
  let context = this.getUIContext().getHostContext() as common.UIAbilityContext;
  let path: string = context.cacheDir;
  // FIX: the sample is `fs.rmdir(path); fs.mkdir(path);` - two un-awaited,
  // un-chained promises with no rejection handler.
  try {
    await fs.rmdir(path);
  } catch (err) {
    Logger.error('rmdir failed: ' + JSON.stringify(err));
  }
  try {
    await fs.mkdir(path);
  } catch (err) {
    Logger.error('mkdir failed: ' + JSON.stringify(err));
  }
}
```

**Two async calls on the same path with no ordering between them is a coin
flip, and one side of the coin breaks the feature.** If `mkdir` reaches the
filesystem before `rmdir` finishes, `mkdir` fails with EEXIST (unhandled
rejection) and `rmdir` then completes - deleting the directory. The app is left
with no `cacheDir`, so the later `fs.openSync(cacheDir + '/result_*.gif',
CREATE | READ_WRITE)` throws ENOENT and GIF generation fails for the whole
session.

The awaited form above is the minimal fix and matches the finding. The better
fix is not to touch `cacheDir` as a whole at all: it is the *application's*
cache directory, shared with the framework and any other component that uses
it, and a page's `aboutToAppear` has no business emptying it. Deleting the
previous `result_*.gif` files by name achieves the same cleanup without the
collateral. Note the sample already names its outputs with
`new Date().getTime()`, so old files are identifiable and never collide.

**The encode — `entry/src/main/ets/views/Module.ets`**
(corrected, see `HW-18-0024`, `HW-18-0005`, `HW-18-0036`)

```typescript
async GifGenerator() {
  // 从图库选择图片
  let pixelMapList: image.PixelMap[] = [];
  let photoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
  photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
  photoSelectOptions.maxSelectNumber = Constants.MAX_SELECT_NUMBER;
  let photoPicker = new photoAccessHelper.PhotoViewPicker();
  let result = await photoPicker.select(photoSelectOptions);
  const uris: Array<string> = result.photoUris;
  if (uris.length === 0) {
    return;                                   // FIX: sample proceeds and writes a 0-byte GIF
  }
  for (let i = 0; i < uris.length; i++) {
    let fileImg = fileIo.openSync(uris[i], fileIo.OpenMode.READ_ONLY);
    const imageSource: image.ImageSource = image.createImageSource(fileImg.fd);
    let decodingOptions: image.DecodingOptions = {
      editable: true,
      desiredPixelFormat: Constants.PIXEL_FORMAT_3,   // RGBA_8888
    };
    try {
      pixelMapList[i] = await imageSource.createPixelMap(decodingOptions);
    } finally {
      imageSource.release();                  // FIX: absent in the sample
      fileIo.closeSync(fileImg);
    }
  }

  // 生成GIF图
  let context = this.getUIContext().getHostContext() as common.UIAbilityContext;
  let path: string = context.cacheDir + '/result_' + new Date().getTime() + '.gif';
  let file = fs.openSync(path, fs.OpenMode.CREATE | fs.OpenMode.READ_WRITE);
  let packer = image.createImagePacker();
  try {
    // 动图编码参数
    let ops: image.PackingOptionsForSequence = {
      frameCount: uris.length,                          // 帧数为选择的图片数
      delayTimeList: [Constants.DELAY_TIME_LIST_50],    // 帧的延迟时间,单位为10ms
      disposalTypes: [Constants.DISPOSAL_TYPES_3],      // 帧过渡模式,3（恢复到之前的状态）
      loopCount: Constants.LOOP_COUNT_0                 // 循环次数为无限循环
    };
    // 合成动图
    await packer.packToFileFromPixelmapSequence(pixelMapList, file.fd, ops);
    Logger.info('Succeeded in packToFileMultiFrames.');
    this.uri = fileUri.getUriFromPath(path);
    this.pathInfos.pushPathByName('show', this.uri);
  } catch (error) {
    Logger.error(JSON.stringify(error));
  } finally {
    packer.release();                         // FIX: absent in the sample
    fs.close(file);
  }
}
```

**Four options in `PackingOptionsForSequence` carry the output, and their units
are the part people get wrong.** `frameCount` must equal the length of the
pixel-map array - it is not a hint, it tells the encoder how many frames to
read. `delayTimeList` is in **hundredths of a second**, so
`DELAY_TIME_LIST_50` is 500 ms per frame, a slow two-frames-per-second GIF, not
50 ms. `disposalTypes: 3` is "restore to previous", the safest choice when
frames may differ in transparency, at the cost of larger output than `1`
("do not dispose") would give for opaque photographs. `loopCount: 0` is
infinite - not "play zero times".

`delayTimeList` and `disposalTypes` are declared as arrays because they are
*per frame*. The sample passes one element for what may be nine frames. It
does not crash - the encoder falls back for the unlisted frames - but it means
the constants can never express a varying-speed animation, which is most of the
reason the API takes lists at all. Build both arrays to `uris.length`.

`desiredPixelFormat: 3` is `RGBA_8888`, and `editable: true` is required
because the encoder needs writable frame buffers. Note the sample also does a
pointless `readSync` of the first 4096 bytes into a throwaway buffer before
creating the image source - dead code, safe to drop.

The empty-selection guard is the difference between a cancel being a no-op and
a cancel producing a zero-byte `result_*.gif` in the cache and navigating to a
preview page showing nothing. `HW-18-0024` records the same missing guard in
five other photography samples.

**The version gate — same file** (as shipped)

```typescript
import { deviceInfo } from '@kit.BasicServicesKit';

@State sdkApiVersionInfo: number = deviceInfo.sdkApiVersion;

Flex({ justifyContent: FlexAlign.SpaceAround }) {
  ForEach(this.entryArr, (item: ModuleItem, index: number) => {
    if (this.sdkApiVersionInfo < Constants.API_18 && index !== Constants.INDEX_FORTH) {
      Column() {
        Image(item.icon).width(24).objectFit(ImageFit.Cover).margin({ bottom: 10 });
        Text(item.name).fontSize(14).height(19).fontWeight(400).fontColor(Color.Black);
      }
      .alignItems(HorizontalAlign.Center).height(50).width('23%');
    } else if (this.sdkApiVersionInfo >= Constants.API_18) {
      Column() {
        Image(item.icon).width(24).objectFit(ImageFit.Cover).margin({ bottom: 10 });
        Text(item.name).fontSize(14).height(19).fontWeight(400)
          .fontColor(index === Constants.INDEX_FORTH ? Color.Blue : Color.Black);
      }
      .onClick(async () => {
        if (index === Constants.INDEX_FORTH) {
          this.GifGenerator();
        }
      })
      .alignItems(HorizontalAlign.Center).height(50).width('23%');
    }
  });
}
.width('100%');
```

**`deviceInfo.sdkApiVersion` is the runtime API level of the device, which is
not the same as the SDK you compiled against.** That is the whole reason this
gate is needed: `compatibleSdkVersion` in the project is `6.0.0(20)`, but the
app is expected to run on API 18 devices too, and
`packToFileFromPixelmapSequence` was introduced at 18 - so above the floor for
this sample but below what a real app's `minAPIVersion` might be. Reading it
into `@State` at construction means the gate is evaluated once and the ForEach
branches consistently.

The trade-off worth naming: **the shipped `ForEach` has no key generator**, and
neither branch is reachable for `INDEX_FORTH` on an old device, so the list
silently changes length with the runtime environment. That is fine here because
`moduleData` is static, but if the module list were data-driven you would want
to filter the array before the `ForEach` rather than branch inside it -
otherwise the index-based `INDEX_FORTH` comparison stops meaning "the GIF tile".

The GIF preview page (`Show.ets`) is worth one note: it displays the animated
result with a plain `Image(this.uri).objectFit(ImageFit.Contain)`. ArkUI's
`Image` animates GIFs natively; no player, no frame loop, no extra component.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`, and none are needed:
`PhotoViewPicker` grants temporary read on the picked URIs, and the
`SaveButton` in `Show.ets` mints its own short-lived write grant for
`createImageAssetRequest` / `applyChanges`.

This sample is one of the two in the industry that got the manifest right;
compare `HW-18-0004` on `PHOTO-10`, where nine sibling samples declare
restricted `READ_IMAGEVIDEO` / `WRITE_IMAGEVIDEO` for materially the same
flows.

`deviceTypes` includes `wearable` alongside `phone`, `tablet` and `2in1`,
which is optimistic for a 460 vp banner and a four-column tile row; the layout
is not adapted for a watch.

`routerMap: "$profile:route_map"` is required, since `Show` is reached by
`pushPathByName('show', ...)`.

## Constraints

- API Version 20 Release or later for the project; HarmonyOS 6.0.0 Release SDK
  or later; DevEco Studio 6.0.0 Release or later.
- `packToFileFromPixelmapSequence` requires **API 18 or later** at runtime.
  This is the constraint the whole `deviceInfo` gate exists to handle.
- All frames are held as decoded pixel maps simultaneously. Nine
  full-resolution photos as RGBA_8888 is on the order of hundreds of megabytes
  before encoding starts - downscale with `desiredSize` for anything beyond a
  demo.
- GIF is limited to 256 colours per frame; photographs will band. The encoder
  quantises silently.
- The output frame size follows the pixel maps. Mixed-resolution inputs are not
  normalised by the encoder, so pre-scale the sequence to one size.
- The 拍摄照片 button and the first three module tiles have no handlers - this
  is a GIF sample wearing a home page.
- `MAX_SELECT_NUMBER` is 9 and `frameCount` is derived from the actual
  selection, so a one-image selection produces a valid single-frame GIF.

## Pitfalls

- **`HW-18-0040`** (B/medium, confirmed): `aboutToAppear` calls
  `fs.rmdir(path); fs.mkdir(path);` on `cacheDir` - both async, both
  un-awaited, un-chained, rejections unhandled. If `mkdir` resolves first it
  fails EEXIST and the subsequent `rmdir` deletes the directory, after which
  `fs.openSync(cacheDir + '/result_*.gif')` throws ENOENT and GIF generation is
  broken for the session. Fix: `await` the removal, then `await` the creation
  (or use the sync forms), with error handling - and prefer deleting only your
  own output files.
- **`HW-18-0024`** (B/low, confirmed): the picker cancel path is unhandled -
  `photoUris` is empty, the decode loop runs zero times, and the code still
  opens a file, writes a zero-byte `result_*.gif` and navigates to an empty
  preview; `GifGenerator()` is also invoked un-awaited from `onClick`, so the
  rejection is unhandled. Six samples share the defect;
  `ImageRotateAndFlip` shows the intended guard. Fix: return early on
  `photoUris.length === 0` and await/catch the call.
- **`HW-18-0005`** (B/low, confirmed): the `ImageSource` created per picked
  image in `Module.ets` is never released, leaking native decoder memory on
  every GIF composed; eight photography files share this. Fix:
  `imageSource.release()` in a `finally` after `createPixelMap`.
- **`HW-18-0036`** (B/low, confirmed): the `ImagePacker` at `Module.ets:120` is
  created per save and never released, and the decoded `pixelMapList` is never
  released either, so each run leaks a packer plus every frame; eight samples
  share this, with `CompressImages` (`PHOTO-09`) the reference implementation.
  Fix: `packer.release()` in a `finally`, and release the pixel maps once the
  pack resolves.

## References

- `documentation/harmonyos-references/04_media/arkts-apis-image-imagepacker.md` - `packToFileFromPixelmapSequence`, `PackingOptionsForSequence`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagesource.md` - `createImageSource`, `DecodingOptions`, `desiredPixelFormat`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagesource
- `documentation/harmonyos-references/03_system/js-apis-device-info.md` - `deviceInfo.sdkApiVersion`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-device-info
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `rmdir`, `mkdir`, `openSync`, `close`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `select`, `PhotoSelectOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-mediaassetchangerequest.md` - `createImageAssetRequest`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-mediaassetchangerequest
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` - `SaveButtonOnClickResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-image.md` - native GIF playback in `Image`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-image
- `PHOTO-09` - the same pick/encode/sandbox/`SaveButton` pipeline, factored into a utility and releasing its packer
- `PHOTO-10` - the anchor card for the restricted-permission systematic this sample correctly avoids
