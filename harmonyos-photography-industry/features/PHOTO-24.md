---
id: PHOTO-24
title: Ratio crop frame - a draggable crop rectangle over an Image, clamped in vp and applied with PixelMap.crop
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/24_image_cropping.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_cropping-0000002426210646
sample: huawei_industry_tree/18_photography/downloads/CropRect.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [componentUtils, fileIo, hilog, image, photoAccessHelper, window]
permissions: [ohos.permission.WRITE_IMAGEVIDEO]
min_api: 20
modules: [entry (entry)]
findings: [HW-18-0070, HW-18-0071, HW-18-0072, HW-18-0022, HW-18-0036, HW-18-0004, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when you need a **crop frame the user can drag over a photo**: the
eight corner and edge handles, the rule-of-thirds guides that fade in while
dragging, the 1:1 / 4:3 / 16:9 ratio buttons, and a save to the gallery. It is
the editor surface between "user picked a photo" and "app has the pixels it
wanted".

The pattern is a coordinate problem wearing a UI costume. The frame lives in vp
in an ArkUI tree; the pixels live in a `PixelMap` sized by the camera. Everything
interesting is the translation between those spaces - a `scaleRatio` from the
displayed size against the decoded size, and a `cropRectClone` recording the
boundary the frame may not escape.

**Read `HW-18-0070` before adopting the drag logic.** The maximum-width clamp
measures the crop view's own root instead of the image behind it, so the frame
can only ever shrink. One `getRectangleById` argument breaks otherwise sound
geometry.

## Feature checklist

- A photo fills a black editing area at 80% width and 60% height, drawn with
  `ImageFit.Contain`, with a crop frame over it initialised to the whole image.
- Eight handles - four corners, four edge midpoints - sit on the frame border,
  each inside a 20 vp hot zone that extends past the visible line.
- Dragging any edge or corner resizes the frame; the two horizontal and two
  vertical thirds lines fade in at 0.6 opacity during the drag.
- The frame cannot leave the image, nor go below 54 vp on either axis.
- Releasing the drag crops immediately and reloads the editor with the result.
- Three ratio chips (1:1, 4:3, 16:9) animate the frame to a centred rectangle of
  that ratio over 300 ms, then crop on animation finish.
- A reset arrow re-decodes the original image and returns the frame to it.
- A `SaveButton` packs the pixel map to JPEG, writes it to the gallery and toasts.

## Architecture

One `entry` module: a page, one view component, a util file, and a small model
layer for the two lazy lists.

```
entry/src/main/ets
├── common/Constants.ets              hot-zone width 20, hot-zone offset 10, minimum edge 54
├── entryability/EntryAbility.ets     full-screen layout, black status bar, loads CropRectPage
├── entrybackupability/
├── model
│   ├── BasicDataSource.ets           IDataSource base + RadioIcon/OperateType + the two static lists
│   ├── CropRect.ets                  @Observed x/y/width/height with right/bottom getters and clone()
│   ├── DragObj.ets                   drag state: origin point, Action (which edges), the rect at press
│   ├── OperateTypeDataSource.ets     the five bottom tools
│   └── RadioDataSource.ets           the three ratio chips
├── pages/CropRectPage.ets            @Entry: top bar, image area, ratio chips, tool row, ratio maths
├── utils/CropUtil.ets                decode, cropImage, packing, SaveButton gallery write
└── view/CropView.ets                 the frame: handles, guides, PanGesture, all four drag directions
```

The documented tree matches the zip file for file, but not method for method -
the 工程目录 block is accurate while the 实现思路 code is not (`HW-18-0072`).

**The design decision worth avoiding** is `CropView` measuring its own root as
the container it must fit inside. `getContainerWidth` reads
`getRectangleById('crop_view_container')`, and that id sits on `CropView`'s own
outer `Column`, whose width is `this.cropRect.width + Constants.CLIP_HOT_ZONE_WIDTH`
- the frame plus 20 vp of hot zone. So the "container" is the frame. The clamp
`containerWidth * 0.85` evaluates to `0.85 * width + 17`, below the current width
for any frame wider than about 113 vp, so the first outward drag of a side edge
snaps the frame in and every later drag snaps it in again (`HW-18-0070`).

The correct shape is the one the variable names already assume: measure the clamp
against something that does not move when the frame moves. The image is already
tagged - `Image(...).id('crop_image')`, measured in `initImageInfoWithActualSize` -
and using that id makes `MAX_WIDTH_RATIO` mean what it reads as: at most 85% of
the picture. The rule generalises - **sample a constraint from the constraining
object, never from the constrained one** - since a bound computed from the value
being bounded is a feedback loop, here a strictly contracting one.

The rest of the structure is sound. `CropRect` is `@Observed` and reaches
`CropView` as `@ObjectLink`, so the drag handlers mutate its four numbers in place
through `setPosition`; `cropRectClone` arrives as `@Prop` and is the outer
boundary for this editing round. The crop is destructive and immediate:
`onActionEnd` crops on release, the page swaps in the new pixel map,
`@Watch('flushPixelMap')` nulls and restores it to force `Image` to reload, and
that reload fires `onComplete` again, re-deriving `scaleRatio` and re-fitting the
frame - which is what makes repeated crops composable.

## Implementation steps

1. **Give the `Image` an id and derive everything from `onComplete`.** Do not use
   the `initImageInfo()` the document shows - it is dead code, never computes
   `scaleRatio`, and its `JSON.parse(JSON.stringify(...))` clone strips the
   `right`/`bottom` getters the drag maths needs (`HW-18-0072`).
2. **Compute `scaleRatio` as the min of the two display/actual ratios**, then
   centre the frame on the letterboxed image - `ImageFit.Contain` leaves bars the
   crop region must not include.
3. **Keep the frame in an `@Observed` class**, hand it to the frame component as
   `@ObjectLink`, and mutate it in place through `setPosition`.
4. **Extend the hot zone beyond the border**: give the frame's wrapper 20 vp of
   extra width and height and position it at `-10` on both axes, so a finger
   within 20 vp of a line still grabs it.
5. **Capture the rect at press time** into `DragObj.downPos` and compute every
   update from that snapshot plus the gesture offset, never from the live rect.
6. **Clamp against the image, not against the frame** (`HW-18-0070`), and clamp
   both axes - the sample clamps maximum width only.
7. **Convert vp to source pixels with `scaleRatio` at crop time**, subtracting
   the image origin first and flooring - `image.Region` takes integer pixels.
8. **Clone before cropping.** `PixelMap.crop` mutates in place; cropping the
   displayed map directly would corrupt the image still on screen.
9. **Save inside the `SaveButton` click**, checking
   `SaveButtonOnClickResult.SUCCESS`; release the `ImagePacker` (`HW-18-0036`),
   close the fd used to stage the decode (`HW-18-0022`), and fix
   `unregisterDataChangeListener` to test `>= 0` (`HW-18-0071`).
10. **Do not declare `WRITE_IMAGEVIDEO`** - the `SaveButton` + `createAsset` flow
    is permission-free by design (`HW-18-0004`).

## Verified snippets

All snippets are from `CropRect.zip`. Corrected forms are marked.

**The real initialisation path — `entry/src/main/ets/pages/CropRectPage.ets`** (as shipped)

```typescript
Stack() {
  Image(this.pixelMap)
    .objectFit(ImageFit.Contain)
    .onComplete((event?: ImageLoadResult) => {
      if (event) {
        this.initImageInfoWithActualSize(event);
      }
    })
    .id('crop_image');

  CropView({
    cropRect: this.cropRect, cropRectClone: this.cropRectClone,
    scaleRatio: this.scaleRatio, onCropComplete: this.onCropComplete.bind(this)
  });
}

async initImageInfoWithActualSize(event: ImageLoadResult) {
  // 获取图片实际像素尺寸
  if (this.pixelMap) {
    this.imageActualWidth = (await this.pixelMap.getImageInfo()).size.width;
    this.imageActualHeight = (await this.pixelMap.getImageInfo()).size.height;
  }

  // 获取组件显示尺寸
  let imageInfo = context.getComponentUtils().getRectangleById('crop_image');
  this.imageDisplayWidth = context.px2vp(imageInfo.size.width);
  this.imageDisplayHeight = context.px2vp(imageInfo.size.height);

  // 计算缩放比例（图片实际像素 -> 显示像素）
  this.scaleRatio = Math.min(
    this.imageDisplayWidth / this.imageActualWidth,
    this.imageDisplayHeight / this.imageActualHeight
  );

  // 计算图片在组件中的实际显示位置（居中显示）
  const displayWidth = this.imageActualWidth * this.scaleRatio;
  const displayHeight = this.imageActualHeight * this.scaleRatio;
  const offsetX = (this.imageDisplayWidth - displayWidth) / 2;
  const offsetY = (this.imageDisplayHeight - displayHeight) / 2;

  // 初始化裁剪框为整个图片的显示区域
  this.cropRect = new CropRect(offsetX, offsetY, displayWidth, displayHeight);
  this.cropRectClone = this.cropRect.clone();
}
```

**Three things carry the design here, and the document shows none of them.**
`onComplete` is the trigger, not `aboutToAppear`: the component rectangle is only
meaningful after layout, and it fires again on every reload, which re-fits the
frame after each crop. `Math.min` of the two ratios is the `ImageFit.Contain` rule
written out. And `offsetX`/`offsetY` are the letterbox bars, so the frame's origin
is not the component's.

The document's `initImageInfo()` is in the file (`CropRectPage.ets:240`) and never
called (`HW-18-0072`). It reads `localOffset` instead of deriving the letterbox,
sets no `scaleRatio`, and clones through `JSON.parse(JSON.stringify(...))` - a
plain object without the `right`/`bottom` getters `dragRight` reads.

**The container measurement and the width clamp — `entry/src/main/ets/view/CropView.ets`** (corrected, see `HW-18-0070`)

```typescript
// 可配置的边距比例（裁剪框最大宽度占容器的比例）
private static readonly MAX_WIDTH_RATIO: number = 0.85;
private containerWidth: number = 0;

private getContainerWidth(): void {
  try {
    // FIX: the sample reads 'crop_view_container' - CropView's own root,
    //      whose width is cropRect.width + CLIP_HOT_ZONE_WIDTH
    const componentInfo = this.getUIContext().getComponentUtils().getRectangleById('crop_image');
    this.containerWidth = this.getUIContext().px2vp(componentInfo.size.width);
  } catch (error) {
    this.containerWidth = 300;
  }
}

private getMaxAllowedWidth(): number {
  return this.containerWidth * CropView.MAX_WIDTH_RATIO;
}

dragRight(delX: number, delY: number, newPosition: CropRect, direction: Action): void {
  let newRight = this.dragObj.downPos.right + delX;

  const maxAllowedWidth = this.getMaxAllowedWidth();
  const maxRightByMaxWidth = this.dragObj.downPos.x + maxAllowedWidth;
  if (newRight > maxRightByMaxWidth) {
    newRight = maxRightByMaxWidth;
  }

  if (newRight > this.cropRectClone.right) {
    newRight = this.cropRectClone.right;
  }
  if (newRight - newPosition.x < Constants.EDGES) {
    newRight = newPosition.x + Constants.EDGES;
  }

  // the shipped code then branches on direction.top / direction.bottom to move
  // the vertical edge too, via topEdges()/bottomEdges(); the plain case is:
  this.cropRect.setPosition(newPosition.x, newPosition.y,
    newRight - newPosition.x, newPosition.height);
}
```

**Three clamps in a fixed order, and the order is the design.** Maximum width
first, then the boundary clamp against `cropRectClone` (the frame may not leave
the image), then the minimum size of `Constants.EDGES` = 54 vp - three handle
widths minus their borders, the smallest frame in which the handles still fit. In
any other order the outer bound could override the minimum and produce a
zero-width region that `cropImage` rejects. Note that `dragTop`/`dragBottom` have
no width-style clamp, so the two axes are constrained differently either way.

**The vp-to-pixel translation — `entry/src/main/ets/utils/CropUtil.ets`** (as shipped)

```typescript
export async function cropImage(
  pixelMap: image.PixelMap | undefined,
  clipRect: CropRect,
  originalRect: CropRect,
  scaleRatio: number = 1
): Promise<image.PixelMap | undefined> {
  if (!pixelMap) {
    hilog.error(0x0000, TAG, 'pixelMap is undefined');
    return undefined;
  }

  try {
    // 计算实际裁剪区域
    const actualX = Math.floor((clipRect.x - originalRect.x) / scaleRatio);
    const actualY = Math.floor((clipRect.y - originalRect.y) / scaleRatio);
    const actualWidth = Math.floor(clipRect.width / scaleRatio);
    const actualHeight = Math.floor(clipRect.height / scaleRatio);

    if (actualWidth <= 0 || actualHeight <= 0) {
      return undefined;                        // degenerate frame: nothing to crop
    }

    const region: image.Region = {
      x: Math.max(0, actualX),
      y: Math.max(0, actualY),
      size: { width: actualWidth, height: actualHeight }
    };

    // 创建副本并裁剪
    const newPixelMap: image.PixelMap = await pixelMap.clone();
    await newPixelMap.crop(region);

    return newPixelMap;
  } catch (err) {
    const error = err as BusinessError;
    hilog.error(0x0000, TAG, `Failed to crop: ${error.message}`);
    return undefined;
  }
}
```

**Subtract the origin, then divide - in that order.** `clipRect.x` is a vp
coordinate inside the `Image` component and `originalRect.x` is where the picture
starts inside it after `Contain` letterboxing, so the subtraction converts a
component coordinate into a picture coordinate, and only then does `scaleRatio`
map it to source pixels. Skip the subtraction and every crop is offset by the
black bar.

`pixelMap.clone()` before `crop` keeps the editor coherent: `crop` modifies in
place, and the map being cropped is the one bound to the on-screen `Image`.
Cloning also makes failure cheap - a bad region rejects on the copy - which is
why returning `undefined` is safe here.

**Decode and save — same file** (corrected, see `HW-18-0022`, `HW-18-0036`)

```typescript
export default async function getPixelMap(component: UIContext) {
  let pixelMap: PixelMap | undefined = undefined;
  const fd = await getResourceFd(component);
  const imageSourceApi = image.createImageSource(fd);
  if (!imageSourceApi) {
    hilog.error(0x0000, TAG, 'imageSourceAPI created failed!');
    return pixelMap;
  }
  pixelMap = await imageSourceApi.createPixelMap({
    editable: true
  });
  imageSourceApi.release();
  fileIo.closeSync(fd);                       // FIX: the sample never closes it
  return pixelMap;
}

async function getResourceFd(component: UIContext) {
  const context = component.getHostContext()!;
  const resourceMgr = context.resourceManager;
  let imageBuffer = await resourceMgr.getMediaContent($r('app.media.shuping').id);
  let filePath = context.cacheDir + '/' + 'shuping.jgp';
  let file = fileIo.openSync(filePath, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
  fileIo.writeSync(file.fd, imageBuffer.buffer);
  return file.fd;
}

export async function packing(pixelMap: image.PixelMap, format: string, quality = 100) {
  // 创建图像编码ImagePacker对象
  const imagePackerApi = image.createImagePacker();
  let packOpts: image.PackingOption = { format, quality };
  try {
    return await imagePackerApi.packToData(pixelMap, packOpts);
  } finally {
    imagePackerApi.release();                 // FIX: absent in the sample
  }
}

export async function savePhotoToGalleryBySaveButton(uiContext: UIContext, imgArrayBuffer: ArrayBuffer,
  saveType = 'jpeg') {
  const context = uiContext.getHostContext();
  let helper = photoAccessHelper.getPhotoAccessHelper(context);
  try {
    // onClick触发后5秒内通过createAsset接口创建图片文件，5秒后createAsset权限收回。
    let uri = await helper.createAsset(photoAccessHelper.PhotoType.IMAGE, saveType);
    // 使用uri打开文件，可以持续写入内容，写入过程不受时间限制
    let file = await fileIo.open(uri, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
    await fileIo.write(file.fd, imgArrayBuffer);
    await fileIo.close(file.fd);
    uiContext.getPromptAction().showToast({ message: $r('app.string.utils_saveimage') });
  } catch (error) {
    const err: BusinessError = error as BusinessError;
    hilog.error(0x0000, 'SAVE_PHOTO', `Failed to save photo. Code is ${err.code}, message is ${err.message}`);
  }
}
```

**The save path is why this app needs no permission.** `SaveButton` is a security
component: the system, not the app, witnesses the tap, and the click result
carries a five-second grant to call `createAsset`. The Chinese comment states the
window exactly - create the asset within five seconds, then write through the
returned uri at leisure, because the fd outlives the grant. Any slow work,
packing included, belongs outside that window.

The two corrections are this industry's standing leaks. `getResourceFd` returns a
raw fd no caller closes, so every decode burns a descriptor for the process
lifetime, and that fd is the only handle to the staged cache file, which is never
removed (`HW-18-0022`). `packing` creates an `ImagePacker` per save and drops it
(`HW-18-0036`); `CompressImages`' `PictureProc.ets` is the one sample that
releases correctly. Both are one-line fixes that matter most in an editor, where
saving is what users repeat.

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": "ohos.permission.WRITE_IMAGEVIDEO",
    "reason": "$string:reason",
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": "inuse"
    }
  }
]
```

**Delete this block** (`HW-18-0004`). `WRITE_IMAGEVIDEO` is a restricted (ACL)
permission ordinary applications cannot ship with, and the sample never requests
it at runtime and never needs it: the only write path is `SaveButton` →
`createAsset`, the permission-free flow this sample exists to demonstrate. The
same declaration appears in nine photography samples, evidently from one shared
`module.json5` template; copying it buys an app-review rejection and nothing else.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. The ability runs full screen
  (`setWindowLayoutFullScreen(true)`) with a black status bar.
- The image is a bundled resource (`app.media.shuping`) staged into `cacheDir`.
  There is no picker; wiring one in means replacing `getPixelMap` and re-deriving
  `scaleRatio` from the new decode.
- Only the first bottom tool does anything. Rotate and flip raise a toast.
- The ratio is not locked after a chip is used: 16:9 crops to 16:9, and the next
  free drag leaves that ratio again.
- Every drag release crops for real; the only undo is the reset arrow, which
  discards all edits.
- Both `LazyForEach` calls key on `JSON.stringify(item)` - harmless across three
  and five static entries, an ArkUI anti-pattern on a mutable list.
- `getContainerWidth` swallows its failure and falls back to a hardcoded 300 vp,
  so a failed measurement leaves the clamp silently wrong rather than absent.

## Pitfalls

- **`HW-18-0070`** (B/medium, confirmed): `CropView.getContainerWidth` measures
  `crop_view_container` - `CropView`'s own root, therefore the frame itself - so
  the 85% clamp evaluates to `0.85 * width + 17` and the frame can only shrink.
  Fix: measure the parent image container (`crop_image`).
- **`HW-18-0071`** (B/low, confirmed): `BasicDataSource.unregisterDataChangeListener`
  tests `pos > 0`, so the listener at index 0 can never be removed and keeps
  notifying a detached component. Fix: `>= 0`.
- **`HW-18-0072`** (D/low, confirmed): the document teaches `initImageInfo()`,
  dead code in the zip; the running path is `Image.onComplete` →
  `initImageInfoWithActualSize()`. Fix: publish the running code.
- **`HW-18-0022`** (B/low, confirmed): systematic across five samples - files
  opened with `fileIo.openSync` are never closed; here `CropUtil.getResourceFd`
  leaks one descriptor per decode. Fix: close it in a `finally`.
- **`HW-18-0036`** (B/low, confirmed): systematic across eight samples - an
  `ImagePacker` is created per save and never released; here in `CropUtil.packing`.
  Fix: `release()` in a `finally`, as `CompressImages` does.
- **`HW-18-0004`** (D/medium, confirmed): systematic across nine samples -
  restricted `WRITE_IMAGEVIDEO` declared although the sample saves exclusively
  through `SaveButton`, which needs no permission. Fix: delete the entry.

## References

- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `PanGesture`, `onActionStart/Update/End`, `fingerList` local vs global coordinates
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-componentutils.md` - `getRectangleById` and `ComponentInfo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-componentutils
- `documentation/harmonyos-references/02_application-framework/ts-explicit-animation.md` - `animateTo` and the `onFinish` the ratio chips crop from
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-explicit-animation
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` - `IDataSource`, listener registration, key generators
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` - `SaveButton`, `SaveButtonOnClickResult`, the temporary authorisation
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `documentation/harmonyos-references/04_media/arkts-apis-image-pixelmap.md` - `crop`, `clone`, `getImageInfo`, `image.Region`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-pixelmap
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagepacker.md` - `packToData` and `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagesource.md` - `createImageSource(fd)`, `createPixelMap`, `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagesource
- `documentation/harmonyos-references/04_media/arkts-apis-photoaccesshelper-photoaccesshelper.md` - `createAsset` and the five-second window
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-photoaccesshelper-photoaccesshelper
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - why `WRITE_IMAGEVIDEO` must not be declared here
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
