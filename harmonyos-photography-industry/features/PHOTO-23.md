---
id: PHOTO-23
title: Mosaic brush - stroke a repeating tile onto a Canvas laid over the photo, then export the composite by component snapshot
industry: 18_photography
doc: huawei_industry_tree/18_photography/docs/23_image_draw_mosaic.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/image_draw_mosaic-0000002413444896
sample: huawei_industry_tree/18_photography/downloads/ImageDrawMosaic.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.ImageKit", "@kit.MediaLibraryKit", "@kit.PerformanceAnalysisKit"]
apis: [Canvas, CanvasRenderingContext2D, RenderingContextSettings, createPattern, ImageBitmap, Path2D, quadraticCurveTo, strokeStyle, globalCompositeOperation, PanGesture, onGestureRecognizerJudgeBegin, GestureJudgeResult, onAreaChange, "UIContext.getComponentSnapshot", SaveButton, SaveButtonOnClickResult, "image.createImagePacker", packToData, "image.createImageSource", createPixelMap, photoAccessHelper, PhotoViewPicker, createAsset, fileIo]
permissions: [ohos.permission.WRITE_IMAGEVIDEO]
min_api: 20
modules: [entry]
findings: [HW-18-0069, HW-18-0036, HW-18-0004, HW-18-0024, HW-18-0092]
status: verified-with-fixes
---

## When to use

Load this card when the user has to **obscure part of a photo before sharing
it** - a face, a licence plate, an account number on a screenshot - by hand
rather than by snapping to a rectangle. The pattern: leave the photo untouched,
stack a transparent `Canvas` over it at the same size, and stroke a repeating
mosaic tile along the pan path. Nothing is composited into the image until the
user saves.

The transferable idea is that **the mosaic is a brush, not a filter**. There is
no per-pixel averaging anywhere in this sample: a 687-byte tile
(`rawfile/mosaic.jpg`) becomes a `CanvasPattern` with `'repeat'`, the pattern
becomes the `strokeStyle`, and an 8vp round-joined line paints it. One texture
upload instead of a pixel loop. The same shape covers any textured brush - a
highlighter, a blur-tile eraser, a sticker trail.

The export half generalises further. The stack carries `.id('mosaic')` and
saving is `getComponentSnapshot().get('mosaic')`, so the system flattens photo
plus strokes and no drawing code is written twice. Use it whenever what you want
to save is what is on screen.

## Feature checklist

- The page opens on a bundled photo decoded from `rawfile/image.jpg`; an add
  button swaps in any image chosen from the gallery.
- Tapping the 马赛克 (mosaic) tool enters edit mode: a title bar with a back
  arrow appears and the drawing canvas is mounted over the photo.
- Dragging a finger paints a mosaic-textured stroke that follows the path
  smoothly, not straight segments.
- Strokes are only accepted while the tool is selected; outside edit mode the
  canvas rejects every gesture but keeps what was already drawn visible.
- 还原 (restore) clears all strokes and returns to the unedited photo.
- Saving is refused with a 未处理图片 (image not processed) toast if nothing has
  been drawn; otherwise photo and strokes are captured as one image and written
  to the gallery as a PNG, reporting 成功保存到相册.

## Architecture

One `entry` module, four source files, 589 lines including the ability. There is
no data layer - the whole document is one `Path2D` and a list of points.

```
entry/src/main/ets
├── entryability/EntryAbility.ets     full-screen layout, dark colour mode, avoid areas -> AppStorage
├── entrybackupability/
├── pages/MainPage.ets                @Entry: the stack, the tool bar, the gestures, the save handler
├── utils/ImageUtil.ets               rawfile decode, gallery pick, uri -> ArrayBuffer
└── viewModel/DrawViewModel.ets       @Observed: the canvas context, the pattern, the path
```

The documented 工程目录 matches the zip exactly, and so do both code blocks in
实现思路 - the `screenShot` method and the `SaveButton` handler are the shipped
code character for character. That is not true of every sample here (compare
`PHOTO-24`, whose document teaches a method the zip never calls, `HW-18-0072`).

**The design decision worth copying** is that `DrawViewModel` owns the
`CanvasRenderingContext2D` and the page merely hands it to the `Canvas`.
`MainPage` declares `@State viewModel: DrawViewModel = new DrawViewModel()` and
writes `Canvas(this.viewModel.context)`; every gesture callback then forwards
two numbers - or nothing on end - and knows nothing about paths, patterns or
composite operations. The drawing model becomes testable without a UI, and the
page keeps only page state: which tool is selected, which pixel map is shown.

The cost of that split is the defect this card exists to warn about. Because the
view model constructs the context, it is tempted to use it in its constructor -
and at construction time no `Canvas` has attached to that context (`HW-18-0069`).
Owning the context is right; touching it before the component is ready is not.

## Implementation steps

1. **Decode the starting image off the resource manager**, writing the bytes to
   `cacheDir` first because `createImageSource` wants an fd or a buffer, not a
   resource id. Close the file and release the `ImageSource` in a `finally` -
   `getPixelMapFromRaw` does both and is the reference form for this industry
   (contrast `HW-18-0022`, `HW-18-0005`).
2. **Guard the picker's cancel path** before indexing `photoUris[0]`
   (`HW-18-0024`).
3. **Stack the canvas on the photo**: an `Image` and a `Canvas` of identical
   `width('100%')` and `height(288)` inside one `Stack`, and give the `Stack` -
   not the canvas - the id you will snapshot.
4. **Mount the canvas conditionally** on `this.isSelected || this.viewModel.hasDrawn`,
   so it is absent before the first stroke and stays mounted afterwards, keeping
   the drawing visible outside edit mode.
5. **Create the pattern in `Canvas.onReady`, never in a constructor**
   (`HW-18-0069`), and surface a null result instead of silently keeping the
   default black `strokeStyle`.
6. **Cache the canvas size from `onAreaChange`** - `clearRect` needs a width and
   a height, and the component reports them nowhere else.
7. **Reject gestures with `onGestureRecognizerJudgeBegin`** rather than by
   unbinding the `PanGesture`, so the canvas stays mounted and visible while
   inert.
8. **Smooth the stroke with `quadraticCurveTo` through midpoints**, not
   `lineTo`, or a fast drag draws a polygon.
9. **Snapshot with `{ scale: 2, waitUntilRenderFinished: true }`** so the export
   is not a stale frame at screen resolution.
10. **Release the `ImagePacker` in a `finally`** after `packToData`
    (`HW-18-0036`), and **do not declare `WRITE_IMAGEVIDEO`** - `SaveButton`
    and `createAsset` inside its handler need no permission (`HW-18-0004`).

## Verified snippets

All snippets are from `ImageDrawMosaic.zip`. Corrected forms are marked.

**The drawing model — `entry/src/main/ets/viewModel/DrawViewModel.ets`**
(corrected, see `HW-18-0069`)

```typescript
import { hilog } from '@kit.PerformanceAnalysisKit';

export class DrawPathPointModel {
  x: number = 0;
  y: number = 0;
}

// Configure brush
export class DrawPathModel {
  public lineWidth: number = 8;
  public img: ImageBitmap = new ImageBitmap('resources/rawfile/mosaic.jpg');
}

@Observed
export class DrawViewModel {
  public hasDrawn: boolean = false;
  public settings: RenderingContextSettings = new RenderingContextSettings(true);
  public context: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings);
  public drawModel: DrawPathModel = new DrawPathModel();
  public canvasHeight: number = 0;
  public canvasWidth: number = 0;
  private pattern: CanvasPattern | null = null;   // FIX: sample assigns this in the constructor
  private points: DrawPathPointModel[] = [];
  private drawPath = new Path2D();

  initPattern() {                                 // FIX: call from Canvas.onReady, not from a constructor
    this.pattern = this.context.createPattern(this.drawModel.img, 'repeat');
    if (!this.pattern) {
      hilog.error(0x0000, 'DrawViewModel', '%{public}s', 'createPattern returned null');
    }
  }

  moveStart(x: number, y: number) {
    this.points.push({ x: x, y: y });
    this.drawPath.moveTo(x, y);
    this.drawCurrentPathModel();
    this.hasDrawn = true;
  }

  moveUpdate(x: number, y: number) {
    let lastPoint = this.points[this.points.length - 1];
    this.points.push({ x: x, y: y });
    this.drawPath.quadraticCurveTo((x + lastPoint.x) / 2, (y + lastPoint.y) / 2, x, y);
    this.drawCurrentPathModel();
    this.hasDrawn = true;
  }

  moveEnd() {
    this.points = [];
    this.drawPath = new Path2D();
    this.hasDrawn = true;
  }

  // clearPath() resets both and calls context.clearRect(0, 0, canvasWidth, canvasHeight),
  // using the size canvasAreaChange(area) caches from the component.

  private drawCurrentPathModel() {
    this.context.globalCompositeOperation = 'source-over';
    this.context.lineWidth = this.drawModel.lineWidth;
    if (this.pattern) {
      this.context.strokeStyle = this.pattern;
    }
    this.context.lineJoin = 'round';
    this.context.stroke(this.drawPath);
  }
}
```

**Three lines carry the whole mosaic effect.** `createPattern(img, 'repeat')`
tiles the 687-byte `mosaic.jpg` in both axes; assigning it to `strokeStyle`
fills the 8vp line with the tile instead of a colour; `lineJoin = 'round'` keeps
the tiling continuous where segments meet. The reference confirms the contract -
`strokeStyle` takes a `CanvasPattern` and defaults to `'#000000'` - which is why
a null pattern degrades to black lines rather than failing loudly.

`moveEnd` resetting `drawPath` to a fresh `Path2D` is what keeps this cheap.
Without it every `stroke()` re-rasterises the whole session's geometry on each
pan update; with it, a finished stroke is baked into the canvas surface and the
live path holds only the current gesture. `points` exists solely to remember the
previous point for the quadratic control point, so it is cleared at the same
moment.

**The stack, the gesture filter and the pan — `entry/src/main/ets/pages/MainPage.ets`**
(corrected, see `HW-18-0069`)

```typescript
Stack() {
  Image(this.isPixelMapChange ? this.pixelMapChanged : this.pixelMap)
    .width('100%')
    .height(288);
  if (this.isSelected || this.viewModel.hasDrawn) {
    Canvas(this.viewModel.context)
      .width('100%')
      .height(288)
      .id('canvas')
      .onReady(() => {
        this.viewModel.initPattern();          // FIX: absent in the sample
      })
      .onAreaChange((oldValue: Area, newValue: Area) => {
        this.viewModel.canvasAreaChange(newValue);
      })
      .onGestureRecognizerJudgeBegin((event: BaseGestureEvent, current: GestureRecognizer,
        others: Array<GestureRecognizer>) => {
        if (!this.isSelected) {
          return GestureJudgeResult.REJECT; // Directly block all gestures
        }
        return GestureJudgeResult.CONTINUE;
      })
      .gesture(
        PanGesture()
          .onActionStart((event: GestureEvent) => {
            let finger: FingerInfo = event.fingerList[0];
            if (finger === undefined) {           // a second finger can reorder the list
              return;
            }
            this.viewModel.moveStart(finger.localX, finger.localY);
          })
          // onActionUpdate is the same shape, calling moveUpdate(finger.localX, finger.localY)
          .onActionEnd((event: GestureEvent) => {
            this.viewModel.moveEnd();
          })
      );
  }
}
.id('mosaic');
```

**The `if` condition and the two ids are the load-bearing parts.** The canvas is
mounted when the tool is selected *or* anything has been drawn: entering edit
mode creates it, leaving edit mode keeps it alive so the strokes remain on
screen, and only `clearPath()` - which flips `hasDrawn` back to false - unmounts
it. The inner `.id('canvas')` identifies the recognizer target; the outer
`.id('mosaic')` on the `Stack` is what gets snapshotted, and it must be the
stack, because a snapshot of the canvas alone would be strokes on transparency.

`onGestureRecognizerJudgeBegin` returning `REJECT` while `isSelected` is false is
what makes "visible but inert" possible - a mounted canvas that draws nothing
because its pan recognizer is refused before it starts. The shipped version also
tests the target id and `GestureControl.GestureType.PAN_GESTURE`, but both paths
return `CONTINUE`, so that test decides nothing; it is trimmed above and is only
worth keeping for a canvas inside a scrolling parent.

**The save handler — same file** (corrected, see `HW-18-0036`)

```typescript
SaveButton({ text: SaveDescription.SAVE })
  .onClick(async (event: ClickEvent, result: SaveButtonOnClickResult) => {
    // toast save_failed and return unless result === SaveButtonOnClickResult.SUCCESS;
    // toast save_disable and return unless this.viewModel.hasDrawn
    let screenshotPixelMap = await this.screenShot('mosaic');
    if (!screenshotPixelMap) {
      return;
    }
    let imagePackerApi: image.ImagePacker = image.createImagePacker();
    try {                                             // FIX: sample never releases the packer
      let packOpts: image.PackingOption = { format: 'image/png', quality: 98 };
      let arrayBuffer = await imagePackerApi.packToData(screenshotPixelMap, packOpts);
      let helper = photoAccessHelper.getPhotoAccessHelper(this.getUIContext().getHostContext()!);
      let uri = await helper.createAsset(photoAccessHelper.PhotoType.IMAGE, 'png');
      let file = await fileIo.open(uri, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
      await fileIo.write(file.fd, arrayBuffer);
      await fileIo.close(file.fd);
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.save_successfully') });
    } finally {
      imagePackerApi.release();
    }
  });
```

**`createAsset` is legal here only because it is inside this handler.** A
`SaveButton` click resolves to a `SaveButtonOnClickResult`, and `SUCCESS` is a
one-shot, system-granted authorisation to write one file to the media library -
which is why the failure string is 设置权限失败 ("failed to set the permission")
rather than "save failed". Calling `createAsset` from any other code path in
this app would fail, and no declared permission would rescue it (`HW-18-0004`).

The snapshot options are chosen, not incidental. `waitUntilRenderFinished: true`
flushes pending draw commands before capturing; without it the reference warns
that "the snapshot captures content rendered in the last frame", which after a
just-finished stroke is the frame before it. `scale: 2` exports at twice the
on-screen 288vp, though captures larger than the screen can fail outright.

**Decoding, done right and done wrong — `entry/src/main/ets/utils/ImageUtil.ets`**
(as shipped)

```typescript
export async function getPixelMapFromRaw(context: Context) {
  // imageBuffer <- resourceMgr.getRawFileContent('image.jpg'), filePath <- cacheDir + '/test.png',
  // decodingOptions <- { editable: true, desiredPixelFormat: RGBA_8888 }
  let file: fileIo.File | undefined = undefined;
  let imageSourceApi: image.ImageSource | undefined;
  try {
    file = fileIo.openSync(filePath, fileIo.OpenMode.READ_WRITE | fileIo.OpenMode.CREATE);
    fileIo.writeSync(file.fd, imageBuffer.buffer);
    imageSourceApi = image.createImageSource(file.fd);
    let pixelMap = await imageSourceApi.createPixelMap(decodingOptions);
    return pixelMap;
  } catch (e) {
    let err = e as BusinessError;
    hilog.error(0x0000, 'ImageUtil', `err message: ${err.message}, err code: ${err.code}`);
  } finally {
    if (file) {
      fileIo.closeSync(file.fd);
    }
    if (imageSourceApi) {
      imageSourceApi.release();       // both native handles freed on every path
    }
  }
  return null;
}

export async function getPixelMapFromGallery() {
  // ... same decodingOptions, then:
  await photoViewPicker.select(photoSelectOptions)
    .then(async (photoSelectResult: photoAccessHelper.PhotoSelectResult) => {
      uris = photoSelectResult.photoUris;
      let imageBuffer = getBufferByUri(uris[0]);          // no length guard: HW-18-0024
      let imageSource: image.ImageSource = image.createImageSource(imageBuffer);
      // ...
      pixelMap = await imageSource.createPixelMap(decodingOptions);
      imageSource.release();
    });
  return pixelMap;
}
```

**`getPixelMapFromRaw` is the resource-hygiene reference for this industry.**
Both native handles - the file descriptor and the `ImageSource` - are declared
outside the `try`, checked in the `finally`, and released there. That is exactly
what `HW-18-0022` (files never closed) and `HW-18-0005` (image sources never
released) find missing in five and eight other photography samples, and why
neither lists this project: `ImageDrawMosaic` gets both right. `editable: true`
is required, not cosmetic - a non-editable pixel map cannot be re-encoded later.

`getPixelMapFromGallery`, immediately below, gets the same job wrong. It indexes
`uris[0]` without checking the array is non-empty, so cancelling feeds
`undefined` into `getBufferByUri`, which returns `null`, which goes into
`createImageSource` - and neither call site in `MainPage` attaches a `.catch`.
Cancelling is routine, not an error; guard on `photoUris.length === 0`
(`HW-18-0024`).

## Permissions & config

```json5
"requestPermissions": [
  {
    "name": 'ohos.permission.WRITE_IMAGEVIDEO',
    "reason": '$string:reason',
    "usedScene": {
      "abilities": ["EntryAbility"],
      "when": 'always'
    },
  }
]
```

**Delete this block.** `WRITE_IMAGEVIDEO` is a restricted (ACL) permission that
ordinary apps cannot ship with, and nothing here needs it: reading is
`PhotoViewPicker`, which needs no permission, and writing is `createAsset`
inside a `SaveButton` click, which carries its own temporary authorisation. No
code path calls `requestPermissionsFromUser`, so the declaration is inert at
runtime and fatal at app review. This is the systematic `HW-18-0004`, whose
evidence lists nine photography samples sharing one `module.json5` template;
this project is a tenth instance it does not name. `when: 'always'` compounds
it - the app has no background task.

`EntryAbility` sets `COLOR_MODE_DARK` unconditionally and the page paints
`backgroundColor(Color.Black)`, so the sample ignores the system theme by design.
Avoid areas go through `AppStorage` and `@StorageProp`, the safer of the two
forms used in this industry.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- The photo area is a fixed `288` vp tall with `margin({ top: 190 })`. Nothing
  derives from the image's aspect ratio, so a portrait photo is stretched into
  the same box as a landscape one and the snapshot preserves that distortion.
- One brush, one size: `lineWidth` is a hardcoded `8` and the tile is a single
  687-byte JPEG, so the mosaic grain does not scale with the image.
- No undo. 还原 is `clearPath()`, which wipes the whole canvas; per-stroke
  history means keeping the finished `Path2D` objects rather than discarding
  them in `moveEnd`.
- After a successful save, `isPixelMapChange` flips to `true` and the `Image`
  switches to the flattened snapshot while the canvas still holds the same
  strokes, so the mosaic is drawn twice over itself until 还原 or a new pick
  resets the state.
- The bundled `rawfile/image.jpg` (2.3 MB) is copied to `cacheDir` as
  `test.png` on each launch and never cleaned up.

## Pitfalls

- **`HW-18-0069`** (B/low, probable) — **the mosaic pattern is created in the
  view model's constructor,** before any `Canvas` has attached to that
  `CanvasRenderingContext2D` (the component enters the tree only once
  `isSelected` is true). If `createPattern` returns null there, the guard in
  `drawCurrentPathModel` skips `strokeStyle` and every stroke is painted default
  black - the effect degrades to a marker pen, unlogged. Fix: create the pattern
  in `Canvas.onReady`, as the reference example does, and report a null result.
- **`HW-18-0036`** (B/low, confirmed) — **`image.createImagePacker()` at
  `MainPage.ets:243` is never released.** Every save leaks a native packer
  instance for the process lifetime. This is a systematic across eight
  photography samples; `CompressImages` `PictureProc.ets` is the one that
  releases correctly. Fix: `imagePackerApi.release()` in a `finally`.
- **`HW-18-0024`** (B/low, confirmed) — **the picker's cancel path is
  unguarded.** `getPixelMapFromGallery` indexes `photoUris[0]` unconditionally
  and neither `onClick` in `MainPage` attaches a `.catch`, so cancelling raises
  an unhandled rejection. The systematic names six samples with this shape; this
  project is a seventh, and `ImageRotateAndFlip` has the correct empty-array
  guard. Fix: return early when `photoUris.length === 0`.
- **`HW-18-0004`** (D/medium, confirmed) — **restricted `WRITE_IMAGEVIDEO` is
  declared although the sample was deliberately built on the permission-free
  `SaveButton` and `PhotoViewPicker` flows.** Nine samples share the defect and
  this one makes ten. Fix: delete the `requestPermissions` entry.

## References

- `documentation/harmonyos-references/02_application-framework/ts-canvasrenderingcontext2d.md` - `createPattern`, `strokeStyle`, `lineJoin`, and the `onReady` example this card's fix follows
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-canvasrenderingcontext2d
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - the `Canvas` component, `onReady` and `onAreaChange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvaspattern.md` - what a `CanvasPattern` is
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvaspattern
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-path2d.md` - `Path2D`, `moveTo`, `quadraticCurveTo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-path2d
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `PanGesture` and `FingerInfo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/ts-gesture-blocking-enhancement.md` - `onGestureRecognizerJudgeBegin` and `GestureJudgeResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-gesture-blocking-enhancement
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-componentsnapshot.md` - `get`, and the stale-frame note behind `waitUntilRenderFinished`; the `SnapshotOptions` table is in `js-apis-arkui-componentsnapshot.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-componentsnapshot
- `documentation/harmonyos-references/02_application-framework/ts-security-components-savebutton.md` - `SaveButton` and `SaveButtonOnClickResult`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-security-components-savebutton
- `documentation/harmonyos-references/04_media/arkts-apis-image-imagepacker.md` - `packToData` and `release`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-image-imagepacker
- `documentation/harmonyos-guides/05_media/photoaccesshelper-savebutton.md` - the save-without-permission flow
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/photoaccesshelper-savebutton
- `documentation/harmonyos-guides/04_system/restricted-permissions.md` - why `WRITE_IMAGEVIDEO` must not be declared here
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/restricted-permissions
- `PHOTO-24` - the other editing surface over the same picker-and-`SaveButton` skeleton
