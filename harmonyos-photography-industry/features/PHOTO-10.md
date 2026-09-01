---
id: PHOTO-10
title: Sticker overlay editor - compose in a Stack, flatten with componentSnapshot, save with SaveButton
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/10_picture_sticker.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/picture_sticker-0000002293373244
sample: huawei_industry_tree/18_photography/downloads/PictureSticker.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [fileIo, hilog, image, photoAccessHelper, window]
permissions: [ohos.permission.WRITE_IMAGEVIDEO]
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0004, HW-18-0005, HW-18-0024, HW-18-0025, HW-18-0036, HW-18-0039, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card for the **"decorate a photo and keep the result" editor** -
stickers, emoji, badges, price tags, signature stamps laid over an image and
dragged into place. The technique generalises well past stickers: anything you
can render in ArkUI on top of an image can be flattened the same way, including
text overlays, drawn annotations, frames and measurement guides.

The pattern is three moves. Compose the editable scene as a `Stack` of
draggable child components over the base `Image`, give that `Stack` an `.id()`.
When the user is done, ask `componentSnapshot.get(id)` for a `PixelMap` of what
is on screen - the framework rasterises the live component tree, so you never
write compositing maths. Then pack that pixel map to PNG and push it into the
album through a `SaveButton`.

**This is the industry's anchor card for `HW-18-0004`.** Nine photography
samples ship a `module.json5` declaring restricted album permissions their code
was explicitly designed not to need. `PictureSticker` is the clearest case:
it reads via `PhotoViewPicker`, writes via `SaveButton`, and still asks for
`WRITE_IMAGEVIDEO`. See **Permissions & config** before you copy anything.

## Feature checklist

- The page opens on a bundled rawfile image, decoded to an editable pixel map.
- A gallery button opens the picker (single image) and replaces the base image.
- A first tap on the sticker circle enters "sticker mode"; a second tap opens
  the sticker tray.
- The tray is a two-lane horizontal `List` of ten sticker icons.
- Tapping a sticker adds a copy at the top-left of the image area.
- Each placed sticker can be dragged with a pan gesture and stays where it is
  released.
- Cancel backs out one level: tray → sticker mode → normal, discarding the
  placed stickers when it leaves the tray.
- 完成 (finished) flattens the composition into a single image.
- The `SaveButton` is enabled only once a flattened image exists, and writes a
  PNG into the album.

## Architecture

One `entry` module. The editable overlay is deliberately a separate component
so each sticker owns its own gesture state.

```
entry/src/main/ets
├── constants/CommonConstants.ets      every layout number, no literals in the layout
├── entryability/EntryAbility.ets      full screen, topAvoid -> AppStorage
├── entrybackupability/
├── pages
│   ├── MainPage.ets                   @Entry: the Stack scene, the tray, the footer, the save
│   └── PicComponent.ets               one draggable sticker (@ObjectLink + PanGesture)
├── utils/ImageUtil.ets                rawfile -> PixelMap, gallery -> PixelMap, uri -> ArrayBuffer
└── viewModel
    ├── ObjectModel.ets                @Observed ObjectModel + ObservedArray<T>
    └── StickerListViewModel.ets       FILTER_STICKER_LIST: ten $r media resources
```

The documented tree matches the zip exactly.

**The design decision worth copying** is the split between `ObjectModel` (the
sticker's committed position, `@Observed`) and `PicComponent`'s own
`@State offsetX/offsetY` (the position during a drag). The pan handler writes
the live offset into local state on every `onActionUpdate`, and only on
`onActionEnd` commits it back through `@ObjectLink` into the model:

```typescript
.onActionUpdate((event: GestureEvent | undefined) => {
  if (event) {
    this.offsetX = this.item.positionX + event.offsetX;   // preview
    this.offsetY = this.item.positionY + event.offsetY;
  }
})
.onActionEnd(() => {
  this.item.positionX = this.offsetX;                     // commit
  this.item.positionY = this.offsetY;
})
```

That is what makes `event.offsetX` - which is cumulative *from the start of the
gesture*, not incremental - correct: it is always added to the committed
origin, never to itself. It also keeps the sixty-times-a-second drag updates
out of the observed array, so only the parent's ten-element `ForEach` reacts to
a finished move, not to every frame.

`ObservedArray<T> extends Array<T>` is the accompanying trick: a bare
`@State Array<ObjectModel>` would not propagate mutations of the elements to
`@ObjectLink` children.

## Implementation steps

1. **Decode a starting image.** From `rawfile` via
   `resourceManager.getRawFileContent`, or from the gallery via
   `PhotoViewPicker`. Ask for `{ editable: true }`.
2. **Close the fd only after `createPixelMap` resolves** (`HW-18-0025`), and
   **release the `ImageSource` when the pixel map is out** (`HW-18-0005`).
3. **Guard the picker cancel path.** A cancelled picker resolves with
   `photoUris = []`, and `uris[0]` is then `undefined` (`HW-18-0024`).
4. **Build the scene as a `Stack`** with the base `Image` first and a `ForEach`
   over the sticker array on top. Give the `Stack` `.id('sticker')` and
   `.clip(true)` so stickers dragged past the edge do not appear in the output.
5. **Model each sticker as `{ id, positionX, positionY, img }`** in an
   `@Observed` class inside an `ObservedArray`, and render it through an
   `@ObjectLink` child component.
6. **Flatten with `getComponentSnapshot().get(id, { scale: 2,
   waitUntilRenderFinished: true })`.** `waitUntilRenderFinished` is what makes
   the capture reflect the last committed drag rather than the previous frame.
7. **Set the "flattened" flag inside the snapshot's `.then`, not before it**
   (`HW-18-0039`).
8. **Save inside the `SaveButton` click:** `packToData` to PNG,
   `helper.createAsset(PhotoType.IMAGE, 'png')`, open, write, close. Release
   the packer afterwards (`HW-18-0036`).
9. **Declare no album permissions** (`HW-18-0004`).

## Verified snippets

All snippets are from `PictureSticker.zip`. Corrected forms are marked.

**Decoding the base image — `entry/src/main/ets/utils/ImageUtil.ets`**
(corrected, see `HW-18-0025`, `HW-18-0005`, `HW-18-0024`)

```typescript
export async function getPixelMapFromRaw(context: Context) {
  let resourceMgr = context.resourceManager;
  let imageBuffer = await resourceMgr.getRawFileContent('image.jpg');
  let filePath = context.cacheDir + '/' + 'test.png';
  let file = fileIo.openSync(filePath, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
  fileIo.writeSync(file.fd, imageBuffer.buffer);
  let imageSourceApi = image.createImageSource(file.fd);
  try {
    return await imageSourceApi.createPixelMap({ editable: true });
  } finally {
    imageSourceApi.release();     // FIX: never released in the sample
    fileIo.closeSync(file.fd);    // FIX: sample closes before createPixelMap runs
  }
}

export async function getPixelMapFromGallery() {
  let photoSelectOptions: photoAccessHelper.PhotoSelectOptions = new photoAccessHelper.PhotoSelectOptions();
  // 过滤选择媒体文件类型为IMAGE_VIDEO_TYPE
  photoSelectOptions.MIMEType = photoAccessHelper.PhotoViewMIMETypes.IMAGE_TYPE;
  // 选择媒体文件的最大数目
  photoSelectOptions.maxSelectNumber = 1;
  let pixelMap: image.PixelMap | undefined = undefined;
  let photoViewPicker = new photoAccessHelper.PhotoViewPicker();
  const photoSelectResult = await photoViewPicker.select(photoSelectOptions);
  const uris: Array<string> = photoSelectResult.photoUris;
  if (uris.length === 0) {
    return undefined;                            // FIX: sample indexes uris[0] unconditionally
  }
  let imageSource: image.ImageSource = image.createImageSource(getBufferByUri(uris[0]));
  if (!imageSource) {
    return undefined;
  }
  try {
    pixelMap = await imageSource.createPixelMap({ editable: true });
  } finally {
    imageSource.release();                       // FIX: never released in the sample
  }
  return pixelMap;
}
```

**The ordering bug in the shipped `getPixelMapFromRaw` is easy to miss and easy
to reproduce.** The sample calls `image.createImageSource(file.fd)`, then
`fileIo.closeSync(file.fd)`, then `await imageSourceApi.createPixelMap(...)`.
An `ImageSource` built over an fd reads from that descriptor lazily, during
decoding - so the descriptor must outlive the `createPixelMap` await, not the
`createImageSource` call. `HW-18-0025` records the same inversion in
`ImageFilter` and `ImageDenoising`. Note that the sibling function
`getPixelMapFromGallery` gets it right by construction: `getBufferByUri`
reads the whole file into an `ArrayBuffer` first, and a buffer-backed
`ImageSource` has no descriptor to lose.

`{ editable: true }` is required here, not decorative: an immutable pixel map
cannot be used as the source of a later pack in some paths, and the pixel map
is the app's working copy for the whole editing session.

**The composition scene — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
Column() {
  Stack() {
    Image(this.isPixelMapChange ? this.pixelMapChanged : this.pixelMap)
      .width(CommonConstants.FULL_SCREEN)
      .height(CommonConstants.IMAGE_HEIGHT);
    ForEach(this.picture_array, (item: ObjectModel) => {
      //展示数组中的图片
      PicComponent({ item: item })
    }, (item: ObjectModel, index: number) => JSON.stringify(item.id) + index);
  }
  .id('sticker')
  .clip(true);
}
.width(CommonConstants.FULL_SCREEN)
.height(CommonConstants.IMAGE_SHOW_HEIGHT)
.padding({ top: CommonConstants.PADDING_TO_HEADER });
```

**`.id('sticker')` and `.clip(true)` are the two attributes that make this a
capture target rather than just a layout.** The id is the only handle
`componentSnapshot.get` accepts, and it must be unique in the page. `clip(true)`
bounds the raster to the image rectangle: without it a sticker dragged past the
edge of the photo would still be inside the snapshot's bounding box, and the
saved PNG would be larger than the picture with a sticker floating in the
margin.

The `Image` source is a ternary between the original and the flattened pixel
map, which is why `isPixelMapChange` has to be set at the right moment - see
the next snippet. The key generator `JSON.stringify(item.id) + index` is
adequate here because `imgID` monotonically increments, so ids never repeat
within a session.

**Flatten and flag — same file** (corrected, see `HW-18-0039`)

```typescript
screenShot(id: string) {
  this.getUIContext()
    .getComponentSnapshot()
    .get(id, { scale: 2, waitUntilRenderFinished: true })
    .then((pixmap: image.PixelMap) => {
      this.pixelMapChanged = pixmap;
      this.isPixelMapChange = true;        // FIX: the sample sets this in the click handler
      setTimeout(() => {
        this.picture_array = [];
      }, 50);
    })
    .catch((err: Error) => {
      hilog.info(0x0000, 'MainPage', 'error: ' + err);
      this.isPixelMapChange = false;       // FIX: absent - a failed snapshot leaves a blank image
    });
}

// in the 完成 (finished) button:
.onClick(() => {
  this.screenShot('sticker');
  this.isAddSticker = false;
  // FIX: `this.isPixelMapChange = true;` was here, one frame too early
});
```

**Two options on `get()` carry the design.** `scale: 2` renders the component
at twice its layout size, so a 283 vp tall editor produces a usably large PNG
instead of a screen-resolution one - the snapshot is a raster of the *layout*,
so output resolution is entirely this parameter's business. `waitUntilRenderFinished:
true` forces the framework to flush pending layout and draw before capturing;
without it, a sticker dropped immediately before pressing 完成 can be missing
from the result.

The `setTimeout(..., 50)` that clears `picture_array` is a deliberate delay:
clearing the overlay array synchronously inside the `.then` would remove the
stickers from the tree the snapshot has only just finished reading. It is a
timing hack rather than a guarantee, but the intent is right - the placed
stickers are now baked into `pixelMapChanged`, so the live ones must go or they
would be composited twice.

The shipped ordering sets `isPixelMapChange = true` synchronously in the click
handler while `pixelMapChanged` is still assigned only inside the promise. The
`Image` ternary therefore binds to `undefined` for at least one frame (a blank
flash), and if the snapshot rejects, the flag stays `true` forever: the editor
shows nothing and the `SaveButton` - which is `enabled(this.isPixelMapChange)` -
becomes live over a `packToData(undefined)` that throws.

**The permission-free save — same file** (corrected, see `HW-18-0036`)

```typescript
SaveButton({ text: SaveDescription.SAVE })
  .enabled(this.isPixelMapChange)
  .onClick(async (_event: ClickEvent, result: SaveButtonOnClickResult) => {
    if (result === SaveButtonOnClickResult.SUCCESS) {
      this.picture_array = [];
      let imagePackerApi: image.ImagePacker = image.createImagePacker();
      let packOpts: image.PackingOption = { format: 'image/png', quality: 100 };
      try {
        this.arrayBuffer =
          await imagePackerApi.packToData(this.isPixelMapChange ? this.pixelMapChanged : this.pixelMap, packOpts);
        let helper = photoAccessHelper.getPhotoAccessHelper(this.context);
        // onClick触发后10秒内通过createAsset接口创建图片文件，10秒后createAsset权限收回。
        let uri = await helper.createAsset(photoAccessHelper.PhotoType.IMAGE, 'png');
        // 使用uri打开文件，可以持续写入内容，写入过程不受时间限制
        let file = await fileIo.open(uri, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
        await fileIo.write(file.fd, this.arrayBuffer);
        await fileIo.close(file.fd);
      } finally {
        imagePackerApi.release();          // FIX: absent in the sample
      }
      this.getUIContext().getPromptAction().showToast(
        { message: $r('app.string.save_successfully'), duration: CommonConstants.DURATION });
    } else {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.save_failed') });
    }
  });
```

**The sample's own comment is the most important line in the file:**
onClick 触发后10秒内通过 createAsset 接口创建图片文件，10秒后 createAsset 权限
收回 - *within 10 seconds of the click, `createAsset` may create the image file;
after 10 seconds the `createAsset` grant is revoked.* That is the entire
security model of a `SaveButton`. Pressing it mints a short-lived, one-shot
permission to create one asset. The `createAsset` call must therefore happen
promptly inside the handler - which is why `packToData` runs before it and the
potentially slow file write runs after it, against a uri that is already valid
and no longer time-limited.

Two consequences worth internalising. First, you cannot hoist this logic into a
`saveImage()` utility that is called later from a promise chain - the grant
will have expired. Second, and this is the point of `HW-18-0004`: because the
grant exists, **no `WRITE_IMAGEVIDEO` declaration is needed or wanted**.

`quality: 100` with `format: 'image/png'` is a no-op - PNG is lossless and
ignores the quality factor - but harmless.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": 'ohos.permission.WRITE_IMAGEVIDEO',
    "reason": '$string:reason',
    "usedScene": { "abilities": ["EntryAbility"], "when": 'always' },
  }
]
```

**Delete this block.** It is the anchor instance of `HW-18-0004`, and the
reasoning applies to every sample in this industry:

- `WRITE_IMAGEVIDEO` and `READ_IMAGEVIDEO` are **restricted (ACL) permissions**.
  An ordinary third-party app cannot ship with them; the declaration alone is
  grounds for an app-review rejection.
- Nothing in `PictureSticker` requests them. There is no
  `requestPermissionsFromUser` call anywhere in the project, so even on a device
  that would grant them, they are never held.
- Nothing in `PictureSticker` needs them. Reading is `PhotoViewPicker`, which
  hands back a temporarily readable URI. Writing is `SaveButton` +
  `createAsset`, which mints its own 10-second grant. Both mechanisms exist
  precisely so that apps do not have to hold album permissions.
- `"when": 'always'` compounds it: a foreground-only photo editor asking for
  permanent background album write.

The same declared-but-unused pattern appears across nine photography samples,
evidently from one shared `module.json5` template: `04_ratio_camera`
(READ + WRITE), `05_image_stitch` (WRITE, saves through a `SaveButton`
`createAsset` window), `09_compress_images` (READ + WRITE), `10_picture_sticker`
(WRITE), `18_image_denoising` (WRITE), `20_image_rotate_and_flip` (WRITE),
`21_video_water_mark` (READ + WRITE), `24_image_cropping` (WRITE) and
`29_camera_twist` (READ + WRITE). `VideoWatermark` even carries a comment on
`FileUtil.ets:39` stating that the picker 不需要特殊权限 (*needs no special
permission*) while its manifest asks for both. **If you copy any photography
sample as a starting point, the first edit is to empty `requestPermissions`,**
then add back only what the code actually calls.

Also note `EntryAbility` writes `topAvoid` into `AppStorage`, and `MainPage`
reads it with `AppStorage.get('topAvoid')` cast through `px2vp` - an untyped
read that yields `undefined` if the ability has not run yet.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `componentSnapshot.get` captures **what is rendered**, so anything off-screen,
  clipped or hidden by `Visibility.None` is absent from the result. It cannot
  capture a component that has never been laid out.
- Output resolution is `layout size × scale`. There is no way to snapshot at
  the base image's native resolution: the saved PNG is a 2× raster of a 283 vp
  tall editor, not a re-composite of the original pixels. For full-resolution
  output you need real pixel-level compositing, not a snapshot.
- Stickers are fixed 50×50 vp with `ImageFit.None`; there is no rotate, no
  scale and no delete-one - cancel discards all of them.
- Placement is always the top-left corner: `handleImageClick` pushes
  `positionX: 0, positionY: 0` regardless of where the tray item was tapped.
- The base image is written to `cacheDir/test.png` on every entry and never
  cleaned up.

## Pitfalls

- **`HW-18-0004`** (D/medium, confirmed): the module declares restricted
  `WRITE_IMAGEVIDEO` although the code saves exclusively through `SaveButton`
  and reads through `PhotoViewPicker` - neither needs any permission, and none
  of the nine affected samples ever requests it at runtime. Fix: delete the
  `requestPermissions` entries from all nine `module.json5` files.
- **`HW-18-0039`** (B/low, confirmed): `isPixelMapChange` flips to `true` in the
  click handler while `pixelMapChanged` is assigned only in the snapshot's
  `.then`, so the `Image` binds to an undefined pixel map for a frame; a failed
  snapshot leaves the flag true, the image blank and `packToData(undefined)`
  throwing behind an enabled `SaveButton`. Fix: set the flag inside `.then`,
  reset it in `.catch`.
- **`HW-18-0025`** (B/low, probable): `getPixelMapFromRaw` closes the fd
  immediately after `createImageSource(file.fd)` and before the async
  `createPixelMap` decode, which can fail the decode on the startup path and
  leave the editor blank; the same inversion is in `ImageFilter` and
  `ImageDenoising`. Fix: move `closeSync` after the awaited `createPixelMap`.
- **`HW-18-0024`** (B/low, confirmed): cancelling the picker resolves with an
  empty `photoUris`, but `ImageUtil.ets` indexes `uris[0]` regardless and the
  caller in `MainPage` attaches no `.catch`, so a routine cancel raises an
  unhandled rejection; six samples share the defect and `ImageRotateAndFlip`
  shows the intended empty-array guard. Fix: return early on
  `photoUris.length === 0` and add `.catch` to the call chain.
- **`HW-18-0005`** (B/low, confirmed): the `ImageSource` instances created at
  `ImageUtil.ets:32` and `:58` are never released, so every image load and every
  gallery pick leaks native decoder memory; eight photography files share this.
  Fix: `imageSource.release()` in a `finally` after `createPixelMap`.
- **`HW-18-0036`** (B/low, confirmed): the `ImagePacker` created at
  `MainPage.ets:213` is never released, leaking a native packer per save;
  eight samples share this, with `CompressImages` (`PHOTO-09`) the one that
  gets it right. Fix: `imagePacker.release()` in a `finally`.

## References

- `documentation/harmonyos-references/02_application-framework/js-apis-arkui-componentsnapshot.md` - `get`, `SnapshotOptions`, `scale`, `waitUntilRenderFinished`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-arkui-componentsnapshot
- `documentation/harmonyos-references/02_application-framework/ts-container-stack.md` - overlay layout and stacking order
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-stack
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` - `SaveButtonOptions`, `SaveButtonOnClickResult`, the temporary grant
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoviewpicker.md` - `PhotoSelectOptions`, `PhotoSelectResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoviewpicker
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `createAsset` and its 10-second window
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagepacker.md` - `packToData`, `PackingOption`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagesource.md` - `createImageSource`, `createPixelMap`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagesource
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `open`, `write`, `close`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `PHOTO-09` - the same `SaveButton` save from a sandbox file, and the correct packer release
- `PHOTO-11` - the same picker-to-`PixelMap` load without the fd-lifetime defect
